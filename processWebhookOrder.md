<?php

namespace App\Jobs;

use App\Integration\EmailMarketingUtil;
use App\Integration\Event\Event;
use App\Integration\Event\EventEmitter;
use App\Integration\Event\OmnisendOrderFulfilledEvent;
use App\Integration\Event\OmnisendOrderPaidEvent;
use App\Integration\Klaviyo;
use App\Integration\Omnisend;
use App\Models\Customer;
use App\Models\Email;
use App\Models\EmailCampaignDetail;
use App\Models\Order;
use App\Models\Shop;
use App\Services\Customer\CustomerService;
use App\Services\Email\EmailService;
use App\Services\EmailCampaignDetail\EmailCampaignDetailService;
use App\Services\IntegrationAttributes\IntegrationAttributesService;
use App\Services\IntegrationProviders\IntegrationProvidersService;
use App\Services\Job\SendEmailService;
use App\Services\Order\OrderService;
use App\Services\Product\ProductService;
use App\Services\Shop\ShopService;
use Exception;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Arr;

class ProcessWebhookOrderJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    private $orderAttr;
    private $shopName;
    private $orderEvent;

    /**
     * @param array  $orderAttr
     * @param string $shopName
     * @param string $orderEvent  Paid | Fulfillment | Archived | Delivered
     */
    public function __construct(array $orderAttr, string $shopName, string $orderEvent)
    {
        $this->orderAttr  = $orderAttr;
        $this->shopName   = $shopName;
        $this->orderEvent = $orderEvent;
    }

    public function handle(
        ShopService $shopService,
        OrderService $orderService,
        CustomerService $customerService,
        SendEmailService $sendEmailService,
        EmailCampaignDetailService $campaignDetailService,
        EmailService $emailService,
        ProductService $productService,
        IntegrationProvidersService $integrationProvidersService,
        IntegrationAttributesService $integrationAttributesService
    ) {
        try {

            /**
             * -------------------------------------------------
             * 1. Validate shop & ability
             * -------------------------------------------------
             */
            $shop = $shopService->getShopByShopName($this->shopName);
            $ability = $shop->subscription()->ability();

            if (!$ability->enabled('email_review_request')) {
                return;
            }

            if (
                $this->orderEvent === Order::DELIVERED &&
                !$ability->enabled('review_request_upon_delivery')
            ) {
                return;
            }

            /**
             * -------------------------------------------------
             * 2. Get customer email
             * -------------------------------------------------
             */
            $customerEmail = $customerService->findCustomerEmailFromOrder($this->orderAttr);

            if (!$customerEmail) {
                return;
            }

            /**
             * -------------------------------------------------
             * 3. Check existing order
             * -------------------------------------------------
             */
            $order = Order::on('mysql_readonly')
                ->where([
                    'shop_id'  => $shop->id,
                    'order_id' => Arr::get($this->orderAttr, 'id')
                ])
                ->first();

            if ($order && !$this->isContinueSending($order, $this->orderEvent)) {
                return;
            }

            /**
             * -------------------------------------------------
             * 4. Save order & customer
             * -------------------------------------------------
             */
            $savedOrder    = $orderService->saveOrder($this->orderAttr, $this->shopName);
            $savedCustomer = $customerService->saveCustomer($this->orderAttr);

            /**
             * -------------------------------------------------
             * 5. Klaviyo Integration
             * -------------------------------------------------
             */
            $canUseKlaviyo = $ability->enabled('klaviyo') ?? false;
            $util = new EmailMarketingUtil(new Klaviyo($shop->id));

            if ($canUseKlaviyo && $util->isConnected() && $savedOrder) {

                if (in_array($this->orderEvent, [Order::PAID, Order::FULFILLMENT])) {

                    $productsByOrder = $productService
                        ->getProductsByOrder($shop, $savedOrder)
                        ->toArray();

                    $event = $this->orderEvent == Order::PAID
                        ? Event::ORDER_PAID
                        : Event::ORDER_FULFILLED;

                    $customerName = "{$savedCustomer->first_name} {$savedCustomer->last_name}";
                    $shopName = explode('.myshopify.com', $shop->shop)[0];

                    $products = array_map(function ($item) use (
                        $savedCustomer,
                        $shop,
                        $customerEmail,
                        $customerName,
                        $shopName,
                        $savedOrder
                    ) {
                        return [
                            'title'      => $item['title'],
                            'first_name' => $savedCustomer->first_name,
                            'last_name'  => $savedCustomer->last_name,
                            'full_name'  => $customerName,
                            'image'      => $item['image'],
                            'link'       => "https://$shop->shop/products/{$item['handle']}",
                            'shop_name'  => $shopName,
                            'email'      => $customerEmail,
                            'order_id'   => $savedOrder->id
                        ];
                    }, $productsByOrder);

                    $emitter = new EventEmitter((object) $products, $event);
                    $util->sendEvent($emitter, $customerEmail, '');
                }
            }

            /**
             * -------------------------------------------------
             * 6. Omnisend Integration
             * -------------------------------------------------
             */
            $canUseOmnisend = $ability->enabled('omnisend') ?? false;
            $utilOmnisend = new EmailMarketingUtil(new Omnisend($shop->id));

            $phone = Customer::where('email', $customerEmail)->first()->phone ?? null;

            if ($canUseOmnisend && $utilOmnisend->isConnected() && $savedOrder && $phone) {

                if (in_array($this->orderEvent, [Order::PAID, Order::FULFILLMENT])) {

                    $event = $this->orderEvent == Order::PAID
                        ? Event::ORDER_PAID
                        : Event::ORDER_FULFILLED;

                    $productsByOrder = $productService
                        ->getProductsByOrder($shop, $savedOrder)
                        ->toArray();

                    $customerName = "{$savedCustomer->first_name} {$savedCustomer->last_name}";
                    $shopName = explode('.myshopify.com', $shop->shop)[0];

                    $products = array_map(function ($item) use (
                        $savedCustomer,
                        $shop,
                        $customerEmail,
                        $customerName,
                        $shopName
                    ) {
                        return [
                            'title'      => $item['title'],
                            'first_name' => $savedCustomer->first_name,
                            'last_name'  => $savedCustomer->last_name,
                            'full_name'  => $customerName,
                            'image'      => $item['image'],
                            'link'       => "https://$shop->shop/products/{$item['handle']}",
                            'shop_name'  => $shopName,
                            'email'      => $customerEmail
                        ];
                    }, $productsByOrder);

                    $emitter = new EventEmitter((object) $products[0], $event);

                    $utilOmnisend->sendTrigger(
                        $emitter,
                        $customerEmail,
                        $phone,
                        null
                    );
                }
            }

            /**
             * -------------------------------------------------
             * 7. Internal Review Request Email
             * -------------------------------------------------
             */
            $reviewRequestEmails = $shop->activeRequestEmailCampaignDetails($this->orderEvent);

            if (
                $reviewRequestEmails->count() <= 0 ||
                !$savedCustomer ||
                !$savedOrder
            ) {
                return;
            }

            $reviewRequestEmail = $campaignDetailService
                ->filterAutomationEmailToSend($shop, $reviewRequestEmails);

            if (!$reviewRequestEmail) {
                return;
            }

            $email = $emailService->createEmailToSend(
                $shop->id,
                $savedCustomer->id,
                $reviewRequestEmail,
                $customerEmail,
                $savedOrder->id
            );

            if (!$email) {
                return;
            }

            $sendEmailService->automationSend(
                $shop,
                $savedCustomer->id,
                $reviewRequestEmail->id,
                $email,
                $savedOrder->id
            );

        } catch (Exception $e) {
            logger()->error("Error ProcessWebhookOrderJob: $e");
        }
    }

    private function isContinueSending(Order $order, $orderEvent): bool
    {
        if (
            ($orderEvent === Order::ARCHIVED && $order->isClosed()) ||
            ($orderEvent === Order::DELIVERED && $order->isDelivered())
        ) {
            return false;
        }

        return true;
    }
}
