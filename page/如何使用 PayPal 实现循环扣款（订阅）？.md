## 起因

业务需求需要集成 PayPal 实现循环扣款功能。然而在搜索引擎上查找相关开发教程时，除了官网外几乎没有找到有用的资料。最终通过阅读 PayPal 官方文档并花费两天时间成功集成。以下是关于如何使用 PayPal 支付接口的总结。

PayPal 目前提供以下几种接口：

1. 通过 **Braintree** 实现 Express Checkout；
2. 创建 App，通过 **REST API** 的方式（目前主流接口方式）；
3. **NVP/SOAP API**（旧接口，不推荐）。

---

## Braintree 接口

**Braintree** 是 PayPal 收购的一家公司，除了支持 PayPal 支付外，还提供了升级计划、信用卡管理、客户信息管理等功能。相比之下，虽然 PayPal 的 **REST API** 也集成了大部分功能，但 PayPal 的 Dashboard 无法直接管理这些信息，而 Braintree 可以，因此更方便。

此外，Braintree 对于使用 Laravel 框架的开发者非常友好，其 **Cashier** 解决方案默认支持 Braintree。然而，遗憾的是 **Braintree 在国内不支持**。

---

## REST API

REST API 是顺应时代发展的产物。如果你熟悉 **OAuth 2.0** 和 **REST API**，那么使用这些接口应该不会有太大困难。

---

## 旧接口

除非 REST API 无法满足需求（如政策限制），否则不推荐使用旧接口。全球都在向 OAuth 2.0 和 REST API 迁移，逆势而行并不明智。

---

## REST API 的介绍

官方的 [API 文档](https://developer.paypal.com/webapps/developer/docs/api/) 对接口和使用方式有详细说明。但直接调用这些 API 可能较为繁琐，建议通过官方提供的 [PayPal-PHP-SDK](https://github.com/paypal/PayPal-PHP-SDK) 快速上手。

在完成第一个示例之前，请确保以下配置已完成：

1. **Client ID** 和 **Client Secret**；
2. **Webhook API**（必须是 HTTPS 且为 443 端口，本地调试建议结合 ngrok）；
3. **Return URL**（同样需满足 HTTPS 要求）。

完成示例后，理解接口分类有助于实现业务需求。以下是接口分类的简要说明：

- **Payments**：一次性支付接口，不支持循环扣款；
- **Billing Plan & Agreements**：订阅功能，支持循环扣款（本文重点）；
- **Vault**：存储信用卡信息；
- **Notifications**：处理 Webhook 信息；
- 其他接口如 **Payouts**、**Sale**、**Order** 等，本文不涉及。

---

## 如何实现循环扣款

实现循环扣款主要分为以下四个步骤：

1. **创建升级计划并激活**；
2. **创建订阅（Agreement）并跳转到 PayPal 网站等待用户同意**；
3. **用户同意后执行订阅**；
4. **获取扣款账单**。

### 1. 创建升级计划

升级计划对应 `Plan` 类。以下是注意事项：

- 创建的计划初始状态为 `CREATED`，需将状态修改为 `ACTIVE` 才能使用；
- `Plan` 必须包含 `PaymentDefinition` 和 `MerchantPreferences`；
- 如果计划包含试用期（`TRIAL`），则必须同时定义常规支付（`REGULAR`）；
- `setSetupFee` 方法设置首次扣款费用，而循环扣款费用由 `Agreement` 定义。

以下是创建并激活计划的代码示例：

php
public function createPlan($param)
{
    $apiContext = $this->getApiContext();
    $plan = new Plan();

    $plan->setName($param->name)
        ->setDescription($param->desc)
        ->setType('INFINITE');

    $paymentDefinition = new PaymentDefinition();
    $paymentDefinition->setName($param->name)
        ->setType($param->type)
        ->setFrequency($param->frequency)
        ->setFrequencyInterval((string)$param->frequency_interval)
        ->setCycles((string)$param->cycles)
        ->setAmount(new Currency(['value' => $param->amount, 'currency' => $param->currency]));

    $merchantPreferences = new MerchantPreferences();
    $merchantPreferences->setReturnUrl(config('payment.returnurl') . "?success=true")
        ->setCancelUrl(config('payment.returnurl') . "?success=false")
        ->setAutoBillAmount("yes")
        ->setSetupFee(new Currency(['value' => $param->amount, 'currency' => 'USD']));

    $plan->setPaymentDefinitions([$paymentDefinition]);
    $plan->setMerchantPreferences($merchantPreferences);

    try {
        $output = $plan->create($apiContext);
        $patch = new Patch();
        $patch->setOp('replace')
            ->setPath('/')
            ->setValue(new PayPalModel('{"state":"ACTIVE"}'));
        $patchRequest = new PatchRequest();
        $patchRequest->addPatch($patch);
        $output->update($patchRequest, $apiContext);
        return $output;
    } catch (Exception $ex) {
        return false;
    }
}


---

### 2. 创建订阅（Agreement）

创建订阅时需注意：

- `setSetupFee` 定义首次扣款费用，`setStartDate` 定义循环扣款的开始时间；
- `setStartDate` 时间格式需为 ISO8601，可使用 Carbon 库处理；
- 使用 `getApprovalLink` 方法获取用户跳转链接。

以下是代码示例：

php
public function createPayment($param)
{
    $apiContext = $this->getApiContext();
    $agreement = new Agreement();

    $agreement->setName($param['name'])
        ->setDescription($param['desc'])
        ->setStartDate(Carbon::now()->addMonths(1)->toIso8601String());

    $plan = new Plan();
    $plan->setId($param['id']);
    $agreement->setPlan($plan);

    $payer = new Payer();
    $payer->setPaymentMethod('paypal');
    $agreement->setPayer($payer);

    try {
        $agreement = $agreement->create($apiContext);
        return $agreement->getApprovalLink();
    } catch (Exception $ex) {
        return "create payment failed, please retry or contact the merchant.";
    }
}


---

### 3. 用户同意后执行订阅

用户同意后需调用 `Agreement` 的 `execute` 方法完成订阅：

php
public function onPay($request)
{
    $apiContext = $this->getApiContext();
    if ($request->has('success') && $request->success == 'true') {
        $token = $request->token;
        $agreement = new Agreement();
        try {
            $agreement->execute($token, $apiContext);
            return $agreement;
        } catch (Exception $e) {
            return null;
        }
    }
    return null;
}


---

### 4. 获取交易记录

订阅后可能不会立即产生交易记录，需稍后再次尝试：

php
public function transactions($id)
{
    $apiContext = $this->getApiContext();
    $params = ['start_date' => date('Y-m-d', strtotime('-15 years')), 'end_date' => date('Y-m-d', strtotime('+5 days'))];
    try {
        $result = Agreement::searchTransactions($id, $params, $apiContext);
        return $result->getAgreementTransactionList();
    } catch (Exception $e) {
        Log::error("get transactions failed" . $e->getMessage());
        return null;
    }
}


---

## 需要考虑的问题

1. 国内使用 Sandbox 测试时连接较慢，需考虑用户中途关闭页面的情况；
2. 必须实现 Webhook，否则用户取消订阅时无法通知网站；
3. 订阅一旦生效，需主动取消旧订阅后再创建新订阅；
4. 订阅切换过程应为原子操作，建议放入队列中执行。

---

👉 [WildCard | 一分钟注册，轻松订阅海外线上服务](https://bit.ly/bewildcard)