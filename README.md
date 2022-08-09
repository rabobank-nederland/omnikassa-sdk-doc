# Software Development Kit
# Rabo OmniKassa

##### _Developer’s manual version_

Version: 1.17 January 2022

Contact e-mail address: contact@omnikassa.rabobank.nl

> © Rabobank, 2022

> No part of this publication may be reproduced in any form by print, photo print, microfilm or any other means without written permission by Rabobank.

> Disclaimer: This manual is exclusively intended for third parties who take care of the technical link between Rabo OmniKassa and the customer's webshop for and on behalf of clients. Rabobank is not liable for possible damage resulting from errors in or incorrect use of the manual. For example, but not limited to, damage that occurs because the technical link between Rabo OmniKassa and the customer's webshop does not work (completely) correctly. Rabobank has the right to change this manual.

### Table of contents

- [Changelog](#changelog)
- [Requirements and installation instructions](#requirements-and-installation-instructions)
  * [PHP SDK](#php-sdk)
  * [Java SDK](#java-sdk)
  * [.NET SDK](#net-sdk)
  * [SSL configuration](#ssl-configuration)
- [Introduction](#introduction)
  * [Purpose](#purpose)
  * [Signing keys and refresh tokens](#signing-keys-and-refresh-tokens)
- [Steps in processing payments](#steps-in-processing-payments)
  * [Order announcement](#order-announcement)
    + [Preparing order](#preparing-order)
    + [Field description](#field-description)
  * [Creating endpoint](#creating-endpoint)
  * [Sending order](#sending-order)
  * [Improve customer experience using payment brand parameters](#payment-brand-parameters)
- [Customer pays order](#customer-pays-order)
- [Receive updates about orders](#receive-updates-about-orders)
- [Request available payment brands](#request-available-payment-brands)
- [Request available iDEAL issuers](#request-available-ideal-issuers)

<a name="changelog"></a>
### Changelog

#### Version 1.17
* Added support for Java 17.

#### Version 1.16
* Added section on retrieval of iDEAL issuers.
* Added section on improving customer experience using payment brand parameters.
* Documented new property `fullName` of CustomerInformation.
* Updated supported .NET versions.
* Various minor corrections and improvements.

<a name="requirements-and-installation-instructions"></a>
### Requirements and installation instructions

<a name="php-sdk"></a>
#### PHP SDK
Install the package through [Composer](https://getcomposer.org/). Edit your project's `composer.json` file by adding:

```json
{
    "require": {
        "rabobank/omnikassa-sdk": "*"
    }
}
```

Next run the Composer update command from the terminal:

    composer update

Then include the composer-generated autoloader.php file in your code using:

```php
require __DIR__ . '/vendor/autoload.php';
```

<a name="java-sdk"></a>
#### Java SDK
If you are using the Java SDK, you should have Java Runtime Environment version 1.8, 11 or 17 available.
To use the Java SDK, extract the ZIP file to the system on which the webshop is installed. The actual SDK is a single jar and is located in the SDK subdirectory.
The jar files that this SDK depends on are in the Lib subdirectory. Both the SDK jar file and the dependent jar files should be included in the classpath to be able to use the Java SDK.

<a name="net-sdk"></a>
#### .NET SDK
The .NET SDK supports the following .NET platforms:

- .NET Framework (version 4.5.2 or higher)
- .NET Core 2.1 / 3.1
- .NET 5.0

You can install the .NET SDK from NuGet https://www.nuget.org/packages/OmniKassa_Rabobank/

If applicable, this document will display separate code examples for these platforms.

<a name="ssl-configuration"></a>
#### SSL configuration
In order to establish a HTTPS connection with Rabo OmniKassa, the Java Runtime environment, .NET environment or PHP (depending on the chosen SDK)
should be configured with a CA bundle so that the certificate of https://betalen.rabobank.nl can be validated.
Normally this is already the case and you do not need to do anything.
If you are running PHP under Windows, you may need to configure the php.ini to include the property openssl.cafile containing the path to the CA bundle. Consult the PHP documentation for more information.

<a name="introduction"></a>
### Introduction

<a name="purpose"></a>
#### Purpose
This document describes how a webshop can be connected to Rabo OmniKassa using the available SDKs.
At this moment Rabobank SDKs provides the following programming languages/platforms: PHP, Java and .NET.
It is possible that this manual already includes functionality that belongs to a payment method that is not yet available in the Rabo OmniKassa dashboard,
this is not a problem for the proper operation of the SDK. In the Rabo OmniKassa Dashboard, the available payment methods can be selected.

<a name="signing-keys-and-refresh-tokens"></a>
#### Signing keys and refresh tokens
In the code, the identifiers {refresh_token} and {signing_key} refer to the refresh token and the signing key respectively.
These indications should not be taken literally but replaced with the values you find in the dashboard of Rabo OmniKassa 2.0.

<a name="steps-in-processing-payments"></a>
### Steps in processing payments
In order to obtain a global picture, this chapter describes the steps in which a payment is made via Rabo OmniKassa.
In the remaining chapters each of these steps is described in more detail.

Each payment consists of the following three steps:

1. **Order announcement**
Before the customer can meet the payment request, the webshop first announces the order at Rabo OmniKassa.
The order will include all the information that Rabo OmniKassa needs to lead the customer through the payment steps.
In a successful order announcement Rabo OmniKassa returns a unique ID identifying the order and an URL that points to the payment pages.
2. **Customer pays the order**
The webshop redirects the customer to the URL that Rabo OmniKassa returned to in the step above.
The customer is directed to Rabo OmniKassa payment pages and can fulfill the payment request.
When this is done, the customer is redirected back to the webshop.
3. **Receive Updates about orders**
In this step, Rabo OmniKassa sends a notification to the webhook URL of the webshop when one or more orders have been processed.
By using a unique token (key) in this notification, the webshop can request the final status of these orders from Rabo OmniKassa.
This step is optional and will only run if the webhook URL is configured in the Rabo OmniKassa dashboard.

The following chapters describe how these steps are implemented using the SDK.

<a name="order-announcement"></a>
#### Order announcement

<a name="preparing-order"></a>
##### Preparing order
In order to make a payment request, an order must first be prepared. The order contains all the information about the
payment request that Rabo OmniKassa needs in order to guide the customer through the payment steps.
The following code blocks are examples for creating an order. The order is subject to requirements and if it does not meet certain criteria,
certain payment methods are not available, the order will be cleaned or rejected.

**PHP**
```php
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\Money;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\OrderItem;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\PaymentBrand;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\PaymentBrandForce;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\PaymentBrandMetaData;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\ProductType;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\request\MerchantOrder;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\Address;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\CustomerInformation;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\VatCategory;

$orderItems = [OrderItem::createFrom([
    'id' => '1',
    'name' => 'Test product',
    'description' => 'Description',
    'quantity' => 1,
    'amount' => Money::fromDecimal('EUR', 99.99),
    'tax' => Money::fromDecimal('EUR', 21.00),
    'category' => ProductType::DIGITAL,
    'vatCategory' => VatCategory::HIGH
])];

$shippingDetail = Address::createFrom([
   'firstName' => 'Jan',
   'middleName' => 'van',
   'lastName' => 'Veen',
   'street' => 'Voorbeeldstraat',
   'postalCode' => '1234AB',
   'city' => 'Haarlem',
   'countryCode' => 'NL',
   'houseNumber' => '5',
   'houseNumberAddition' => 'a'
]);

$billingDetails = Address::createFrom([
   'firstName' => 'Jan',
   'middleName' => 'van',
   'lastName' => 'Veen',
   'street' => 'Factuurstraat',
   'postalCode' => '2314BA',
   'city' => 'Haarlem',
   'countryCode' => 'NL',
   'houseNumber' => '15'
]);

$customerInformation = CustomerInformation::createFrom([
    'emailAddress' => 'jan.van.veen@gmail.com',
    'dateOfBirth' => '20-03-1987',
    'gender' => 'M',
    'initials' => 'J.M.',
    'telephoneNumber' => '0204971111',
    'fullName' => 'Jan van Veen'
]);

$order = MerchantOrder::createFrom([
    'merchantOrderId' => '100',
    'description' => 'Order ID: 100',
    'orderItems' => $orderItems,
    'amount' => Money::fromDecimal('EUR', 99.99),
    'shippingDetail' => $shippingDetail,
    'billingDetail' => $billingDetail,
    'customerInformation' => $customerInformation,
    'language' => 'NL',
    'merchantReturnURL' => 'http://localhost/',
    'skipHppResultPage' => true,
    'paymentBrand' => PaymentBrand::IDEAL,
    'paymentBrandForce' => PaymentBrandForce::FORCE_ONCE,
    'paymentBrandMetaData' => PaymentBrandMetaData::createFrom([
        'issuerId' => 'RABONL2U',
    ])
]);
```

**Java**
```java
import java.util.Collections;
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.*;
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.order_details.*;
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.enums.*;

OrderItem orderItem = new OrderItem.Builder()
    .withId("1")
    .withQuantity(1)
    .withName("Test product")
    .withDescription("Description")
    .withAmount(Money.fromEuros(EUR, new BigDecimal("99.99")))
    .withTax(Money.fromEuros(EUR, new BigDecimal("21.00")))
    .withItemCategory(ItemCategory.DIGITAL)
    .withVatCategory(VatCategory.HIGH)
    .build();

CustomerInformation customerInformation = new CustomerInformation.Builder()
    .withTelephoneNumber("0204971111")
    .withInitials("J.M.")
    .withGender("M")
    .withEmailAddress("jan.van.veen@gmail.com")
    .withDateOfBirth("20-03-1987")
    .withFullName("Jan van Veen")
    .build();

Address shippingDetails = new Address.Builder()
    .withFirstName("Jan")
    .withMiddleName("van")
    .withLastName("Veen")
    .withStreet("Voorbeeldstraat")
    .withHouseNumber("5")
    .withHouseNumberAddition("a")
    .withPostalCode("1234AB")
    .withCity("Haarlem")
    .withCountryCode(CountryCode.NL)
    .build()

Address billingDetails = new Address.Builder()
    .withFirstName("Jan")
    .withMiddleName("van")
    .withLastName("Veen")
    .withStreet("Factuurstraat")
    .withHouseNumber("5")
    .withHouseNumberAddition("a")
    .withPostalCode("1234AB")
    .withCity("Haarlem")
    .withCountryCode(CountryCode.NL)
    .build()

MerchantOrder order = new MerchantOrder.Builder()
    .withMerchantOrderId("ORDID123")
    .withDescription("An example description")
    .withOrderItems(Collections.singletonList(orderItem))
    .withAmount(Money.fromEuros(Currency.EUR, new BigDecimal("99.99")))
    .withCustomerInformation(customerInformation)
    .withShippingDetail(shippingDetails)
    .withBillingDetail(billingDetails)
    .withLanguage(Language.NL)
    .withMerchantReturnURL("http://localhost/")
    .withSkipHppResultPage(true)
    .withPaymentBrand(PaymentBrand.IDEAL)
    .withPaymentBrandForce(PaymentBrandForce.FORCE_ONCE)
    .withPaymentBrandMetaData(Collections.singletonMap("issuerId", "RABONL2U"))
    .build();
```

**.NET**
```csharp
using System.Collections.Generic;
using OmniKassa.Model;
using OmniKassa.Model.Enums;
using OmniKassa.Model.Order;

OrderItem orderItem = new OrderItem.Builder()
   .WithId("1")
   .WithQuantity(1)
   .WithName("Test product")
   .WithDescription("Description")
   .WithAmount(Money.FromEuros(Currency.EUR, 99.99m))
   .WithTax(Money.FromEuros(Currency.EUR, 21.00m))
   .WithItemCategory(ItemCategory.PHYSICAL)
   .WithVatCategory(VatCategory.LOW)
   .Build();

CustomerInformation customerInformation = new CustomerInformation.Builder()
   .WithTelephoneNumber("0204971111")
   .WithInitials("J.M.")
   .WithGender(Gender.M)
   .WithEmailAddress("jan.van.veen@gmail.com")
   .WithDateOfBirth("20-03-1987")
   .WithFullName("Jan van Veen")
   .Build();

Address shippingDetails = new Address.Builder()
   .WithFirstName("Jan")
   .WithMiddleName("van")
   .WithLastName("Veen")
   .WithStreet("Voorbeeldstraat")
   .WithHouseNumber("5")
   .WithHouseNumberAddition("a")
   .WithPostalCode("1234AB")
   .WithCity("Haarlem")
   .WithCountryCode(CountryCode.NL)
   .Build();

 Address billingDetails = new Address.Builder()
   .WithFirstName("Jan")
   .WithMiddleName("van")
   .WithLastName("Veen")
   .WithStreet("Factuurstraat")
   .WithHouseNumber("5")
   .WithHouseNumberAddition("a")
   .WithPostalCode("1234AB")
   .WithCity("Haarlem")
   .WithCountryCode(CountryCode.NL)
   .Build();

Dictionary<string, string> paymentBrandMetaData = new Dictionary<string, string>();
paymentBrandMetaData.Add("issuerId", "RABONL2U");

MerchantOrder order = new MerchantOrder.Builder()
   .WithMerchantOrderId("ORDID123")
   .WithDescription("An example description")
   .WithOrderItems(new List<OrderItem>() { orderItem })
   .WithAmount(Money.FromEuros(Currency.EUR, 99.99m))
   .WithCustomerInformation(customerInformation)
   .WithShippingDetail(shippingDetails)
   .WithBillingDetail(billingDetails)
   .WithLanguage(Language.NL)
   .WithMerchantReturnURL("http://localhost/")
   .WithSkipHppResultPage(true)
   .WithPaymentBrand(PaymentBrand.IDEAL)
   .WithPaymentBrandForce(PaymentBrandForce.FORCE_ALWAYS)
   .WithPaymentBrandMetaData(paymentBrandMetaData)
   .Build();
```

<a name="field-description"></a>
##### Field description
Below are all the fields with the name, a description, and the rules to which the value of the field must comply.

**MerchantOrder**

| Field                  | Description                                                                                                           | Rules                                                                                                                                                                                                                                                                                              |
|----------------------- | --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `merchantOrderId`      | The identity of the order                                                                                             | Required                                                                                                                                                                                                                                                                                           |
|                        |                                                                                                                       | May consist only of alphanumeric characters                                                                                                                                                                                                                                                        |
|                        |                                                                                                                       | This field is shortened to a maximum of 24 characters.                                                                                                                                                                                                                                             |
|                        |                                                                                                                       | For AfterPay, this field must be unique                                                                                                                                                                                                                                                            |
| `description`          | A description of the order                                                                                            | Optional                                                                                                                                                                                                                                                                                           |
|                        |                                                                                                                       | This field is shortened to a maximum of 35 characters.                                                                                                                                                                                                                                             |
| `orderItems`           | All items that will be ordered by the customer                                                                        | Optional                                                                                                                                                                                                                                                                                           |
|                        |                                                                                                                       | A maximum of 99 items may be supplied                                                                                                                                                                                                                                                              |
| `amount`               | The total order amount in cents, including VAT                                                                        | Required                                                                                                                                                                                                                                                                                           |
|                        |                                                                                                                       | The amount must not exceed 99,999.99                                                                                                                                                                                                                                                               |
|                        |                                                                                                                       | The amount must be equal to the sum of all order items of the piece price (including VAT) multiplied by the number of copies.                                                                                                                                                                      |
|                        |                                                                                                                       | Due to the way VAT is calculated, rounding differences can prevent them from being equal.                                                                                                                                                                                                          |
|                        |                                                                                                                       | We therefore recommend that you base the total VAT amount on the VAT of the item price instead of the total order amount excluding VAT.                                                                                                                                                            |
|                        |                                                                                                                       | For example: Suppose the price of an order item (excluding VAT) is €12.98 and a vat rate of 21% is used. When a customer order 7 copies, the price including VAT €12.98 + 21% = €15.71 is completed. The total order amount (including VAT) that is given in this field is 7 x €15.71 = €109.97.   |
|                        |                                                                                                                       | **Note** If the amount is not equal to the sum of the amounts of the order items then                                                                                                                                                                                                              |
|                        |                                                                                                                       | 1. The order items are filtered from the order announcement, and                                                                                                                                                                                                                                   |
|                        |                                                                                                                       | 2. AfterPay is not possible as a payment method                                                                                                                                                                                                                                                    |
| `shippingDetails`      | The delivery address of this order                                                                                    | Optional                                                                                                                                                                                                                                                                                           |
| `language`             | The texts on the payment pages will be in this language                                                               | Optional, the default value is EN.                                                                                                                                                                                                                                                                 |
|                        |                                                                                                                       | ISO 3166-1 Alpha-2, limited to NL, EN, FR and DE.                                                                                                                                                                                                                                                  |
|                        |                                                                                                                       | If an unknown value is used: then EN will be used.                                                                                                                                                                                                                                                 |
| `merchantReturnURL`    | The URL the customer will return to after the payment steps have been completed                                       | Required                                                                                                                                                                                                                                                                                           |
|                        |                                                                                                                       | Must be a valid URL                                                                                                                                                                                                                                                                                |
| `paymentBrand`         | The payment method to which the customer is limited                                                                   | Optional                                                                                                                                                                                                                                                                                           |
|                        |                                                                                                                       | Must be one of the following values: IDEAL, AFTERPAY, PAYPAL, MasterCard, VISA, BANCONTACT, MAESTRO, V_PAY, SOFORT                                                                                                                                                                                         |
|                        |                                                                                                                       | When value is CARDS, then all card payment methods are offered (MasterCard, Visa, Bancontact, Maestro, and V PAY)                                                                                                                                                                                  |
| `paymentBrandForce`    | Is used to enforce the payment method                                                                                 | Optional, if the field `paymentBrand` is supplied then this field is required.                                                                                                                                                                                                                     |                                                                      |
|                        |                                                                                                                       | Must be one of the following values: FORCE_ONCE or FORCE_ALWAYS                                                                                                                                                                                                                                    |
| `paymentBrandMetaData` | Additional parameters specific to the supplied payment method                                                         | Optional, when this field is supplied then the fields `paymentBrand` and `paymentBrandForce` must be set as well.                                                                                                                                                                                  |
|                        |                                                                                                                       | The value of this field is a key-value map of strings.                                                                                                                                                                                                                                             |
| `customerInformation`  | A limited set of customer information                                                                                 | Optional                                                                                                                                                                                                                                                                                           |
| `billingDetails`       | The billing address of this order                                                                                     | Optional                                                                                                                                                                                                                                                                                           |
| `initiatingParty`      | An ID identifying the party from which the order announcement was initiated                                           | Optional. This field must be left empty unless agreed otherwise with Rabobank                                                                                                                                                                                                                      |
| `skipHppResultPage`    | Use this field to skip the hosted result page (also referred to as the success/thank you page) in the payment process | Optional

For more information on how to use the `paymentBrand`, `paymentBrandForce`, `paymentBrandMetaData` and the
`skipHppResultPage` please consult
[Improve customer experience using payment brand parameters](#payment-brand-parameters).

**Money**

| Field      | Description    | Rules                                |
|----------- | -------------- | -------------------------------------|
| `currency` | The currency   | should not be null                   |
|            |                | Must be EUR                          |
|            |                | Other currencies are not supported   |
| `amount`   | The amount     | should not be null                   |
|            |                | Must be a natural number             |

To create a _Money_ instance, use the following code:

**PHP**
```PHP
//1,75 Euro
$moneyFromCents = Money::fromCents('EUR', 175);
//75,99 Euro
$moneyFromDecimal = Money::fromDecimal('EUR', 75.99);

//Rounding examples
//1000 Euro cents
$amountInCents = Money::fromDecimal('EUR', 9.999)->getAmount();
//999 Euro cents
$amountInCents = Money::fromDecimal('EUR', 9.991)->getAmount();
//1000 Euro cents
$amountInCents = Money::fromDecimal('EUR',9.995)->getAmount();
```

**Java**
```java
//1,75 Euro
Money moneyFromEuros = Money.fromEuros(Currency.EUR, BigDecimal.valueOf(175, 2));
Money moneyFromString = Money.fromEuros(Currency.EUR, new BigDecimal("1.75"));
```

**.NET**
```csharp
using OmniKassa.Model;
using OmniKassa.Model.Enums;

Money moneyFromDecimal = Money.FromEuros(Currency.EUR, 1.75m);
```

**Address / ShippingDetails**

| Field                  | Description                          | Rules                                                                                                                   |
|----------------------- | ----------------------------------   | ------------------------------------------------------------------------------------------------------------------------|
| `firstName`            | customer's name                      | Required                                                                                                                |
|                        |                                      | Has a maximum length of 50 Characters                                                                                   |
| `middleName`           | customer's middle name               | Optional                                                                                                                |
|                        |                                      | Has a maximum length of 20 Characters                                                                                   |
| `lastName`             | Last name of the customer            | Required                                                                                                                |
|                        |                                      | Has a maximum length of 50 Characters                                                                                   |
| `street`               | Street name of the address           | Required                                                                                                                |
|                        |                                      | Has a maximum length of 100 Characters.                                                                                 |
|                        |                                      | Note: In the case of card payment at Worldline, this field is shortened to a maximum of 50 characters.                  |
| `postalCode`           | Postal Code of the address           | Required                                                                                                                |
|                        |                                      | Must match the ZIP/Postal code format of the selected CountryCode                                                       |
| `city`                 | City of Address                      | Required                                                                                                                |
|                        |                                      | Has a maximum length of 40 Characters                                                                                   |
| `countryCode`          | Country Code of Address              | Required                                                                                                                |
|                        |                                      | Must be one of the following values:                                                                                    |
|                        |                                      | AF, AX,AL, DZ, VI, AS, AD, AO, AI, AQ, AG, AR, AM, AW, AU, AZ, BS, BH, BD, BB, BE,,BZ, BJ, BM, BT, BO, BQ,              |
|                        |                                      | BA, BW, BV, BR, VG, IO, BN, BG, BF, BI, KH, CA, CF, CL, CN, CX, CC, CO, KM , CG, CD, CK, CR, CU, CW, CY, DK,            |
|                        |                                      | DJ, DM, DO, DE, EC,,EG, SV, GQ, ER, EE, ET, FO, FK, FJ, PH, FI, FR, TF, GF, PF, GA, GM, GE, GH,,GI, GD, GR,             |
|                        |                                      | GL, GP, GU, GT, GG, GN, GW, GY, HT, HM, HN , HU, HK, IE, IS, IN,,ID, IQ, IR, IL, IT, CI, JM, JP, YE, you, JO,           |
|                        |                                      | KY, CV, CM, KZ, KE, KG, KI, UM,,KW, HR, LA, LS, LV, LB, LR, LY, LI, LT, LU, MO, MK, MG, MW, MV, MY, ML, MT, IM,         |
|                        |                                      | MA, MH , MQ, MR, MU, YT, MX, FM, MD, MC, MN, ME, MS, MZ, MM, NA, NR, NL,,NP, NI, NC, NZ, NE, NG, NU, MP, KP, NO,        |
|                        |                                      | NF, UG, UA, UZ, OM, AT, TL, PK, PW, PS, PA, PG, PY, PE, PN, PL, PT, PR, QA, RE , RO, RU, RW, BL, KN, LC, PM, VC,        |
|                        |                                      | SB, WS, SM, SA, ST, SN, RS, SC, SL, SG, SH, MF, SX, SI, SK, SD, SO, ES, SJ, LK, SR, SZ, SY, TJ, TW, TZ, TH, TG, TK,     |
|                        |                                      | TO, TT, TD, CZ, TN, TR, TM, TC, TV ,,UY, VU, VA, VE, AE, US, GB, VN, WF, EH, BY, ZM, ZW, ZA, GS, KR, SS, SE, CH         |
| `houseNumber`          | House number of the address          | Optional                                                                                                                |
|                        |                                      | Has a maximum length of 100 characters.                                                                                 |
|                        |                                      | Note: In the case of card payment at Worldline, this field is shortened to a maximum of 50 characters.                  |
| `houseNumberAddition`  | House number addition of the address | Optional                                                                                                                |
|                        |                                      | Has a maximum length of 6 characters                                                                                    |

The supported ZIP code formats are as follows. The remaining country codes only have a maximum length of 10 and are indicated in the table below as 'Other'.

| Country code | Format                   | Maximum length |
|--------------|--------------------------|----------------|
| BE           | \p{Digit} +              | 4              |
| DE           | \p{Digit} +              | 5              |
| NL           | \p{Digit}{4}\p{Alpha}{2} | 6              |
| Other        | N/a.                     | 10             |

**CustomerInformation**

| Field             | Description                            | Rules                                                                   |
|------------------ | -------------------------------------- | ------------------------------------------------------------------------|
| `emailAddress`    | The email address of the customer      | Optional                                                                |
|                   |                                        | Must be a valid email address                                           |
|                   |                                        | Has a maximum length of 45 characters.                                  |
| `dateOfBirth`     | The date of birth of the customer      | Optional                                                                |
|                   |                                        | Must be in the format: DD-MM-YYYY                                       |
| `gender`          | customer Sex                           | Optional                                                                |
|                   |                                        | Must be one of the following values: M, F                               |
| `initials`        | customer Initials                      | Optional                                                                |
|                   |                                        | May consist only of alphabet characters                                 |
|                   |                                        | Has a maximum length of 256 characters.                                 |
| `telephoneNumber` | The telephone number of the customer   | Optional                                                                |
|                   |                                        | May consist of numbers and alphabet characters                          |
|                   |                                        | Has a maximum length of 31 characters.                                  |
| `fullName`        | The full name of the customer          | Optional                                                                |
|                   |                                        | Has a maximum length of 128 characters.                                 |
|                   |                                        | Consult [this section](#customer-name-dashboard) for more information.  |

For the payment method _AfterPay_, Rabo OmniKassa shows a page to the customer in which he can complete or modify the above fields after the order announcement.
Your webshop will **not** be informed of the details that are filled in during this process.

**OrderItem**

| Field         | Description                                    | Rules                                                              |
|-------------- | ---------------------------------------------- | ------------------------------------------------------------------ |
| `id`          | The ID of the item                             | Optional                                                           |
|               |                                                | Has a maximum length of 25 Characters                              |
| `name`        | Name of the item                               | Required                                                           |
|               |                                                | May consist only of alphabet characters                            |
|               |                                                | Has a maximum length of 50 Characters                              |
| `description` | A description of the item                      | Optional                                                           |
|               |                                                | Has a maximum length of 100 Characters                             |
| `quantity`    | Number of copies                               | Required                                                           |
|               |                                                | Must be a natural number greater than 0                            |
| `amount`      | The amount per piece in cents, including VAT   | Should not be null                                                 |
| `tax`         | VAT per piece in cents                         | Optional                                                           |
| `category`    | Category of this item                          | Should not be null                                                 |
|               |                                                | Must be one of the following values:                               |
|               |                                                | DIGITAL, PHYSICAL                                                  |
| `vatCategory` | VAT Category of the item                       | Optional                                                           |
|               |                                                | Must be one of the following values:                               |
|               |                                                | The values refer to the different rates used in the Netherlands:   |
|               |                                                | 1, 2, 3, 4                                                         |
|               |                                                | 1 = high 21%                                                       |
|               |                                                | 2 = low (9%)                                                       |
|               |                                                | 3 = zero (0%)                                                      |
|               |                                                | 4 = none (exempt from VAT)                                         |

The quantity describes how much the customer wants to have the of a particular product, for example 2 apples,
or 3 pears. The amount indicates how much one product costs, so 1 apple costs €1.50, or 1 pear costs €1.75, and
includes VAT.

If we detail the examples then it becomes:

**PHP**
```php
$apples = OrderItem::createFrom([
    'id' => '1',
    'name' => 'Apple',
    'description' => 'A delicious apple',
    'quantity' => 1,
    'amount' => Money::fromDecimal('EUR', 1.50),
    'tax' => Money::fromDecimal('EUR', 0.14),
    'category' => ProductType::PHYSICAL,
    'vatCategory' => VatCategory::LOW
]);

$pears = OrderItem::createFrom([
    'id' => '2',
    'name' => 'Pear',
    'description' => 'A delicious pear',
    'quantity' => 3,
    'amount' => Money::fromDecimal('EUR', 1.75),
    'tax' => Money::fromDecimal('EUR', 0.16),
    'category' => ProductType::PHYSICAL,
    'vatCategory' => VatCategory::LOW
]);
```

**Java**
```java
OrderItem apples = new OrderItem.Builder()
    .withId("1")
    .withQuantity(2)
    .withName("Apple")
    .withDescription("A delicious apple")
    .withAmount(Money.fromEuros(Currency.EUR, new BigDecimal("1.50")))
    .withTax(Money.fromEuros(Currency.EUR, new BigDecimal("0.14")))
    .withItemCategory(ItemCategory.PHYSICAL)
    .withVatCategory(VatCategory.LOW)
    .build();

OrderItem pears = new OrderItem.Builder()
    .withId("2")
    .withQuantity(3)
    .withName("Pear")
    .withDescription("A delicious pear")
    .withAmount(Money.fromEuros(Currency.EUR, new BigDecimal("1.75")))
    .withTax(Money.fromEuros(Currency.EUR, new BigDecimal("0.16")))
    .withItemCategory(ItemCategory.PHYSICAL)
    .withVatCategory(VatCategory.LOW)
    .build();
```

**.NET**
```csharp
OrderItem apples = new OrderItem.Builder()
   .WithId("1")
   .WithQuantity(2)
   .WithName("Apple")
   .WithDescription("A delicious apple")
   .WithAmount(Money.FromEuros(Currency.EUR, 1.50m))
   .WithTax(Money.FromEuros(Currency.EUR, 0.14m))
   .WithItemCategory(ItemCategory.PHYSICAL)
   .WithVatCategory(VatCategory.LOW)
   .Build();

OrderItem pears = new OrderItem.Builder()
   .WithId("2")
   .WithQuantity(3)
   .WithName("Pear")
   .WithDescription("A delicious pear")
   .WithAmount(Money.FromEuros(Currency.EUR, 1.75m))
   .WithTax(Money.FromEuros(Currency.EUR, 0.16m))
   .WithItemCategory(ItemCategory.PHYSICAL)
   .WithVatCategory(VatCategory.LOW)
   .Build();
```

**Discount**

A discount in the order is also provided by an order item but the amount and (if applicable) the tax fields have a negative value.
The following examples describe a discount of 10 euros:

**PHP**
```php
$discount = OrderItem::createFrom([
    'id' => '1',
    'name' => 'Discount',
    'description' => 'Frequent buyer',
    'quantity' => 1,
    'amount' => Money::fromDecimal('EUR', -10),
    'tax' => Money::fromDecimal('EUR', -0.9),
    'category' => ProductType::PHYSICAL,
    'vatCategory' => VatCategory::LOW
]);
```

**Java**
```java
OrderItem discount = new OrderItem.Builder()
    .withId("1")
    .withQuantity(1)
    .withName("Discount")
    .withDescription("Frequent buyer")
    .withAmount(Money.fromEuros(Currency.EUR, new BigDecimal("-10")))
    .withTax(Money.fromEuros(Currency.EUR, new BigDecimal("-0.9")))
    .withItemCategory(ItemCategory.PHYSICAL)
    .withVatCategory(VatCategory.LOW)
    .build();

```

**.NET**
```csharp
OrderItem discount = new OrderItem.Builder()
   .WithId("1")
   .WithQuantity(1)
   .WithName("Discount")
   .WithDescription("Frequent buyer")
   .WithAmount(Money.FromEuros(Currency.EUR, -10m))
   .WithTax(Money.FromEuros(Currency.EUR, -0.9m))
   .WithItemCategory(ItemCategory.PHYSICAL)
   .WithVatCategory(VatCategory.LOW)
   .Build();
```

The `Amount` of the `MerchantOrder` must match the sum of the order items if the `OrderItems` is supplied.

The total amount can be calculated as follows:

**PHP**
```php
$orderTotalInCents = 0;
foreach ($orderItems as $orderItem) {
    $priceTaxInclusiveInCents = $orderItem->getAmount()->getAmount();
    $orderTotalInCents += $priceTaxInclusiveInCents * $orderItem->getQuantity();
}
$orderAmount = Money::fromCents('EUR', $orderTotalInCents);
```

**Java**
```java
BigDecimal orderTotal = BigDecimal.ZERO;
for (OrderItem orderItem : orderItems) {
    BigDecimal amount = orderItem.getAmount().getAmount();
    BigDecimal quantity = new BigDecimal(orderItem.getQuantity());
    orderTotal = orderTotal.add(amount.multiply(quantity));
}
Money orderAmount = Money::fromEuros(Currency.EUR, orderTotal);
```

**.NET**
```csharp
decimal totalAmount = 0.0m;

foreach (var orderItem in orderItems)
{
    decimal amount = orderItem.Amount.Amount;
    int quantity = orderItem.Quantity;
    totalAmount += quantity * amount;
}
Money orderAmount = Money.FromEuros(Currency.EUR, totalAmount);
```

Should there be an incorrect order which was sent and Rabo OmniKassa can repair it, then an attempt is made to correct the order by removing or shortening any data.
This applies only in extreme cases and may result in specific payment methods not being available for this order.

**Required fields per payment method**

Regardless of the payment method, at least the `merchantOrderId`, `amount` and the `merchantReturnUrl` must be included in each order.
Depending on the payment method, additional information must be included in the order.

The table below shows what data this is, broken down by payment method:

| Payment method | Additional information in the order                                                                                                                                             |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| iDEAL          | No additional information is required for this payment method.                                                                                                                  |
| PayPal         | Although not mandatory, we also recommend that you include order items in the order for additional security regarding PayPal's payment to the webshop owner.                    |
| Bancontact     | Although not mandatory, we advise you to include the delivery address (or, if not known, the billing address) in the order for additional certainty about a successful payment. |
| Visa           | Although not mandatory, we advise you to include the delivery address (or, if not known, the billing address) in the order for additional certainty about a successful payment. |
| MasterCard     | Although not mandatory, we advise you to include the delivery address (or, if not known, the billing address) in the order for additional certainty about a successful payment. |
| V PAY          | Although not mandatory, we advise you to include the delivery address (or, if not known, the billing address) in the order for additional certainty about a successful payment. |
| Maestro        | Although not mandatory, we advise you to include the delivery address (or, if not known, the billing address) in the order for additional certainty about a successful payment. |
| AfterPay       | For AfterPay, the following additional information is required:                                                                                                                 |
|                | - Order items with per order item the ID, the description and the sales tax amount or the sales tax category.                                                                   |
|                | - Billing or delivery address (if different then both addresses are required).                                                                                                  |
|                | In addition, AfterPay requires that the MerchantOrderId field to be unique.                                                                                                     |
|                | The order amount must be at least 5 euro.                                                                                                                                       |
| Sofort         | No additional information is required for this payment method.                                                                                                                                                                                |

If for a payment method the mandatory additional information is missing in the order then the customer cannot use this method to fulfill the payment.
If the payment method was included in the order using the PaymentBrand field, the announcement will be refused by Rabo OmniKassa.

<a name="creating-endpoint"></a>
#### Creating endpoint
Besides an order is also an instance of an `Endpoint` needed. With this `Endpoint` it is possible to perform all calls towards the Rabo OmniKassa.
An `Endpoint` is instantiated with three parameters: an Url, A _Base64_ encoded signing key, and a `TokenProvider` implementation.

The URL determines whether the production environment or the sandbox environment is linked:

| Environment   | Description                                                                                                                                                              |
|-------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Production    | In this environment actual payments are done that are depreciated from the customer's account and transferred to the owner of the webshop.                               |
| Sandbox       | This is a test environment. Payments in this environment are simulated and are only intended to test the connections with your webshop.                                  |

The environments can be configured in Rabo OmniKassa Dashboard - for example, which payment methods should be offered.
The signing key is a secret key that can be found in the Rabo OmniDashboard. This signing key is _BASE64_ encoded.
The signing key is used by Rabo Omnikassa to cryptographically sign the data in the `merchantReturnUrl` as well as the data in the messages that are sent to the webhook.
The webshop must use this information to verify the authenticity of the message. This is done automatically by the SDK..

Finally, we need the implementation of the `TokenProvider`. This implementation is responsible for providing and storing token information.
For example, an implementation can store the data in a database, on a file system, or in memory.
The aim is that your `TokenProvider` provides a refresh token by default -it must be available in the `TokenProvider` before the first call is made.
This token can be found in the Rabo OmniKassa Dashboard and is a unique token with long term validity (typically a few years). It is used to request an access token from Rabo Omnikassa.
The access token is valid for a limited amount of time (typically one hour) and must be used in all subsequent calls. This token is delivered to the `TokenProvider` for storage and also for reading.

**PHP**
```php
<?php

use nl\rabobank\gict\payments_savings\omnikassa_sdk\connector\TokenProvider;

class InMemoryTokenProvider extends TokenProvider
{
    private $map = array();

    /**
     * Construct the in memory TokenProvider with the given refresh token.
     * @param string $refreshToken The refresh token used to retrieve the access tokens with.
     */
    public function __construct($refreshToken)
    {
        $this->setValue(static::REFRESH_TOKEN, $refreshToken);
    }

    /**
     * Retrieve the value for the given key.
     *
     * @param string $key
     * @return string Value of the given key or null if it does not exists.
     */
    protected function getValue($key)
    {
        return array_key_exists($key, $this->map) ? $this->map[$key] : null;
    }

    /**
     * Store the value by the given key.
     *
     * @param string $key
     * @param string $value
     */
    protected function setValue($key, $value)
    {
        $this->map[$key] = $value;
    }

    /**
     * Optional functionality to flush your systems.
     * It is called after storing all the values of the access token and can be used for example to clean caches or reload changes from the database.
     */
    protected function flush()
    {
    }
}
```

**Java**
```java
public class InMemoryTokenProvider extends TokenProvider {
    private Map<FieldName, String> map = new HashMap<>();

    public InMemoryTokenProvider(String refreshToken) {
        setValue(FieldName.REFRESH_TOKEN, refreshToken);
    }

    @Override
    public String getValue(FieldName key) {
        return map.get(key);
    }

    @Override
    public void setValue(FieldName key, String value) {
        map.put(key, value);
    }
}
```

**.NET**
```csharp
using System;
using System.Collections.Generic;
using OmniKassa;

namespace MyNamespace
{
    public sealed class InMemoryTokenProvider : TokenProvider
    {
        private Dictionary<FieldName, String> mMap = new Dictionary<FieldName, String>();

        public InMemoryTokenProvider(String refreshToken)
        {
            SetValue(FieldName.REFRESH_TOKEN, refreshToken);
        }

        protected override String GetValue(FieldName key)
        {
            mMap.TryGetValue(key, out string value);
            return value;
        }

       protected override void SetValue(FieldName key, String value)
        {
            if (mMap.ContainsKey(key))
            {
                mMap[key] = value;
            }
            else
            {
                mMap.Add(key, value);
            }
        }
    }
}
```

This provider can then be used to create an Endpoint as follows:

**PHP**
```php
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\Environment;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\signing\SigningKey;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\endpoint\Endpoint;

$signingKey = new SigningKey(base64_decode('{signing_key}'));
$inMemoryTokenProvider = new InMemoryTokenProvider('{refresh_token}');
$endpoint = Endpoint::createInstance(Environment::PRODUCTION, $signingKey, $inMemoryTokenProvider);
```

**Java**
```java
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.*;
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.connector.TokenProvider;

TokenProvider inMemoryTokenProvider = new InMemoryTokenProvider("{refresh_token}");
Endpoint endpoint = Endpoint.createInstance(Environment.PRODUCTION, "{signing_key}", inMemoryTokenProvider);
```

**.NET**
```csharp
using OmniKassa;
using MyNamespace;

TokenProvider inMemoryTokenProvider = new InMemoryTokenProvider("{refresh_token}");
Endpoint endpoint = Endpoint.Create(Environment.SANDBOX, "{signing_key}", inMemoryTokenProvider);
```

To link to the sandbox environment, the constant `Environment.SANDBOX` must be used as the first parameter. Don't forget to also replace the refresh token and the signing key that are specific to the sandbox environment.

<a name="sending-order"></a>
#### Sending order
With this `Endpoint` we can announce the order. As a response to the order announcement Rabo OmniKassa returns a unique ID to identify the order as well as a URL to the payment pages. The customer must be redirected to this URL in order to pay for the order.

**PHP**
```php
$response = $endpoint->announce($order);

//Redirect user to Rabo OmniKassa
header('Location: ' . $response->getRedirectUrl());
```

**Java**
```java
MerchantOrderResponse response = endpoint.announce(order);
// Redirect user to Rabo OmniKassa
return "redirect:" + response.getRedirectUrl();
```

**.NET Standard**
```csharp
using Microsoft.AspNetCore.Mvc;

MerchantOrderResponse merchantOrderResponse = await endpoint.Announce (order);
String redirectUrl = response.RedirectUrl;
return new RedirectResult(redirectUrl);
```

**.NET Framework**
```csharp
using System.Web.Mvc;

MerchantOrderResponse merchantOrderResponse = endpoint.Announce (order);
String redirectUrl = response.RedirectUrl;
return new RedirectResult(redirectUrl);
```

The object of type `MerchantOrderResponse` as returned by the Java SDK contains the following fields:

| Field            | Description                                                                                                                                                                                    |
|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| redirectUrl      | The URL to which the customer must be redirected to in order to pay for the order.                                                                                                             |
| omnikassaOrderId | A unique ID that Rabo OmniKassa will assign to the order. This ID is needed to link the payment result as returned by the webhook notification mechanism to the correct order (see [Receive updates about orders](#receive-updates-about-orders)). |


<a name="payment-brand-parameters"></a>
#### Improve customer experience using payment brand parameters
The default behavior for an online payment is to redirect the customer to the payment pages of Rabo OmniKassa after the
order has been announced. These are Rabobank-branded pages hosted by Rabo OmniKassa and they guide the customer through
the payment process by presenting a list of payment brands and to ask the customer for additional details, depending on
the selected payment brand. For an improved online payment experience Rabo OmniKassa provides means to skip some of
these steps by letting the webshop supply payment brand information. For example, in the checkout process the webshop
can already provide an option for the customer to select iDEAL as a payment brand as well as the bank. In this section
we describe how the SDK facilitates this improved experience.

The first improvement is to allow the customer to select the payment brand in the webshop. For this the SDK provides
functionality to retrieve the payment brands that are currently configured and active within Rabo OmniKassa for the web
shop. For a technical description on how to obtain the brands see
[Request available payment brands](#request-available-payment-brands). The returned payment brand information can then
be used for example to populate a select box in the webshop.

Once the customer has selected a payment brand, the webshop needs to supply two additional fields in the order
announcement:

1. The `paymentBrand` field must contain the name of the selected payment brand, for example `IDEAL`.
2. The `paymentBrandForce` field must specify whether the customer can select an alternative payment in the hosted
   payment pages.

We explain the payment method and the force options in more detail. When the payment method is `IDEAL` and the Force
option `FORCE_ONCE` has been specified, then the customer will immediately start an iDEAL payment upon arrival at
Rabo OmniKassa and thus arrives on the bank selecting screen. The customer then has the option to finalize the payment
or to choose another payment method by clicking on `<Choose Other Payment method>`. With the `FORCE_ALWAYS` it is not
possible for the customer to choose another payment method. The only options are to approve or cancel the payment after
selecting the bank.

The second improvement applies only to iDEAL and allows the customer to also select his or her bank in the webshop. For
this the SDK provides functionality to retrieve the list of participating banks. For a technical description on how to
obtain this list please consult [Request available iDEAL issuers](#request-available-ideal-issuers). The returned list
of banks can then be used to populate another select box in the webshop that becomes visible after the customer has
selected iDEAL as payment brand.

Once the bank has been selected the webshop needs to include the ID of the bank in the `paymentBrandMetaData` field in
addition to the above-mentioned `paymentBrand` the `paymentBrandForce` fields. The `paymentBrandMetaData` field is a
key-value map. To specify the bank include an entry in this map with key 'issuerId' and as value the ID of the bank as
specified in the issuer list, for example 'RABONL2U'.

By implementing these 2 improvements in the webshop the customer will be immediately redirected to the iDEAL page of
the selected bank after announcing the order to authorize the transaction.

A final optimization can be achieved by setting the `skipHppResultPage` field in the order announcement to `true`. After
authorizing the transaction in the iDEAL page of the selected bank, the customer will be immediately redirected back to
the webshop. The success page (also referred to as the “Thank You” page) of the hosted payment pages will be skipped in
this case.

<a name="customer-name-dashboard"></a>
### How to expose the name of the customer in Omni Dashboard
In this section we specify how to include the name of the customer in the order information such that it will be visible
in the Omni Dashboard. There are two ways on how to achieve this.

1. The first and preferred approach is to specify the field `fullName` of the `CustomerInformation` structure.
2. If this field is not set then OmniKassa will use the name as specified in the shipping details (if included) or billing details (if included), where the shipping details will take precedence.

<a name="customer-pays-order"></a>
### Customer pays order

When redirected to the payment pages of Rabo Omnikassa the customer can pay the order. When this is done, the customer is automatically redirected back to the webshop via the `merchantReturnUrl` specified in the order announcement.
With this `URL` some query parameters will be included that relate to the order, namely:

| Parameter   | Description                                                                     |
|------------ | --------------------------------------------------------------------------------|
| `order_id`  | This is the order ID as specified in the order announcement.                    |
| `status`    | The current known status of the order.                                          |
| `signature` | The signature makes it possible to verify that the request is authentic.        |

To verify that the signature is authentic, the following code can be used.
It is also advisable to make use of the `PaymentCompletedResponse` class so that the `order_id`, `status` and `signature` will be cleaned if they contain invalid/unsafe values.

**PHP**
```php
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\signing\SigningKey;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\response\PaymentCompletedResponse;

$orderId = $_GET['order_id'];
$status = $_GET['status'];
$signature = $_GET['signature'];
$signingKey = new SigningKey(base64_decode('{signing_key}'));
$paymentCompletedResponse = PaymentCompletedResponse::createInstance($orderId, $status, $signature, $signingKey);
if (!$paymentCompletedResponse) {
    throw new Exception('The payment completed response was invalid.');
}

// Use these variables instead of using the URL parameters ($orderId and $status). Input validation has been performed on these values.
$validatedMerchantOrderId = $paymentCompletedResponse->getOrderID();
$validatedStatus = $paymentCompletedResponse->getStatus();

// ... complete payment
```

**Java**
```java
void paymentCompleted(HttpServletRequest request) {
    byte[] signingKey = Base64Utils.decodeFromString("{signing_key}");
    String orderId = request.getParameter("order_id");
    String status = request.getParameter("status");
    String signature = request.getParameter("signature");
    PaymentCompletedResponse paymentCompletedResponse;
    try {
       paymentCompletedResponse = PaymentCompletedResponse.newPaymentCompletedResponse(orderId, status, signature, signingKey);
    } catch (RabobankSdkException invalidSignatureException){
       throw new IllegalStateException("The payment completed response was invalid.", invalidSignatureException);
    }

    // Use these variables instead of using the URL parameters (orderId and status). Input validation has been performed on these values.
    String validatedOrderId = paymentCompletedResponse.getOrderID();
    String validatedStatus = paymentCompletedResponse.getStatus();

   //... complete payment
}
```

**.NET Standard**
```csharp
using System.Collections.Generic;
using Microsoft.Extensions.Primitives;
using Microsoft.AspNetCore.WebUtilities;
using OmniKassa.Model.Response;
using OmniKassa.Model.Enums;

Dictionary<String, StringValues> query = QueryHelpers.ParseQuery(Request.QueryString.Value);
Dictionary<String, String> dictionary = GetDictionary(query);

PaymentCompletedResponse response = PaymentCompletedResponse.Create(dictionary, "{signing_key}");

String validatedOrderId = paymentCompletedResponse.OrderId;
PaymentStatus validatedStatus = paymentCompletedResponse.Status;
```

**.NET Framework**
```csharp
using OmniKassa.Model.Response;
using OmniKassa.Model.Enums;

PaymentCompletedResponse response = PaymentCompletedResponse.Create(Request.QueryString, "{signing_key}");

String validatedOrderId = paymentCompletedResponse.OrderId;
PaymentStatus validatedStatus = paymentCompletedResponse.Status;
```

If the answer is invalid, it is advisable to redirect the customer to an error page but to consider the order as `open`.
At a later stage, the actual status of the order can be requested by means of notifications.

<a name="receive-updates-about-orders"></a>
### Receive updates about orders

 All the status transitions of an order are tracked by the Rabo OmniKassa so that they can be offered to the webshop with the help of notifications.
 The Rabo OmniKassa does this by sending a notification to the `webhookUrl` through a `Post` request with the notification as JSON.
 The `webhookUrl` can be configured in the dashboard of Rabo OmniKassa.

 A notification looks like this:

**Notification**
```json
{
  "authentication": "eyJraWQiOiJHS0wiLCJhbGciOiJFUzI1NiJ9.eyJwayMiOjUwMiwiY2lkIjoiYzhjYy1mMThjIiwiZXhwIjoxNDc5MTIyODc2fQ.MEUCIQC2Z5WUVTAKcBHISsOVMJIJE8PAbVe5x1ior4bgrTcgCwIgLNoVIWEmSbQekJTccM89sosAY-8JzN47DGjvdPGdF0w",
  "expiry": "2016-11-25T09:53:46.765+01:00",
  "eventName": "merchant.order.status.changed",
  "poiId": "123"
}
```

With this JSON it is possible to build up an `AnnouncementResponse` with which the actual information of the orders can be retrieved.

**PHP**
```php
use nl\rabobank\gict\payments_savings\omnikassa_sdk\endpoint\Endpoint;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\Environment;

$signingKey = new SigningKey(base64_decode('{signing_key}'));
$inMemoryTokenProvider = new InMemoryTokenProvider('{refresh_token}');
$endpoint = Endpoint::createInstance(Environment::PRODUCTION, $signingKey, $inMemoryTokenProvider);

$json = file_get_contents('php://input');
$announcementResponse = new AnnouncementResponse($json, $signingKey);

do {
    $response = $endpoint->retrieveAnnouncement($announcementResponse);
    $results = $response->getOrderResults();
    foreach($results as $result) {
        //... Update the order status using the properties in $result
    }
} while ($response->isMoreOrderResultsAvailable());
```

**Java**
```java
@RequestMapping(value = "/webhook", method = RequestMethod.POST)
ResponseEntity webhook(@RequestBody ApiNotification apiNotification) throws RabobankSdkException {
    byte[] signingKey = Base64Utils.decodeFromString("{signing_key}");
    apiNotification.validateSignature(signingKey);
    Endpoint endpoint = Endpoint.createInstance(...);

    MerchantOrderStatusResponse merchantOrderStatusResponse;
    do {
        merchantOrderStatusResponse = endpoint.retrieveAnnouncement(apiNotification);
        for (MerchantOrderResult result : merchantOrderStatusResponse:getOrderResults()) {
            // Update the order status using the properties in result
        }
    } while (merchantOrderStatusResponse.moreOrderResultsAvailable());

    return new ResponseEntity(HttpStatus.OK);
}
```

**.NET Standard**
```csharp
using Microsoft.AspNetCore.Mvc;
using OmniKassa.Model.Response;
using OmniKassa.Model.Response.Notification;

[HttpPost]
public ActionResult Webhook([FromBody] ApiNotification notification)
{
    MerchantOrderStatusResponse response;
    do
    {
       response = await omniKassa.RetrieveAnnouncement(notification);
        foreach(MerchantOrderResult result in response.OrderResults)
        {
            // Update the order status using the properties in result
        }
    }
    while (response.MoreOrderResultsAvailable);

   return new OkObjectResult("");
}
```

**.NET Framework**
```csharp
using System.Web.Mvc;
using OmniKassa.Model.Response;
using OmniKassa.Model.Response.Notification;

[HttpPost]
public ActionResult Webhook(ApiNotification notification)
{
    MerchantOrderStatusResponse response;
    do
    {
        response = omniKassa.RetrieveAnnouncement(notification);
        foreach(MerchantOrderResult result in response.OrderResults)
        {
            // Update the order status using the properties in result
        }
    }
    while (response.MoreOrderResultsAvailable);

    return new HttpStatusCodeResult(HttpStatusCode.OK);
}
```

This step also verifies the signature of the notification.
If the validation fails, for example when a different signing key is active in the dashboard, then the order data will not be retrieved. Rabo Omnikassa will send a new notification at a later time.
More information regarding this process can be found in the User Manual of Rabo OmniKassa.

The `MerchantOrderStatusResponse` object returned by the call to the `retrieveAnnouncement` method consists of the following properties:

| Field                       | Description                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `orderResults`              | This is an array consisting of 0 or more objects of type MerchantOrderResult. The result of an order is included in each object. The composition of this item is further explained in further detail in this section.                                                                                                                                                                                                                         |
| `moreOrderResultsAvailable` | When calling `retrieveAnnouncement`, the data of up to 100 objects is returned in OrderResults, to keep the volume of the response manageable. If more than 100 orders have been processed, the MoreOrderResultsAvailable field has a value of true and a new call to RetrieveAnnouncement can query the results of the next set of up to 100 orders. This process can be repeated as long as moreOrderResultsAvailable has a value of false. |

As shown above, an object of type `MerchantOrderResult` contains the data of a single order. The following table explains the fields of this item.

| Field                 | Description                                                                                                                                                                                            |
|---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `merchantOrderId`     | The ID of the order as specified by the webshop when the order was announced.                                                                                                                          |
| `omniKassaOrderId`    | The unique ID assigned by OmniKassa to the order at the time of the announcement.                                                                                                                      |
| `poiId`               | The unique ID that identifies the webshop.                                                                                                                                                             |
| `orderStatus`         | The status of the order. The following values are possible:                                                                                                                                            |
|                       | COMPLETED: The full amount of the order is paid by the customer.                                                                                                                                       |
|                       | CANCELLED: The order was canceled by the customer.                                                                                                                                                     |
|                       | EXPIRED: The order has expired without the customer having paid.                                                                                                                                       |
| `orderStatusDateTime` | The most recent timestamp of the order.                                                                                                                                                                |
|                       | If there was no payment attempt, this field will contain the time the order was announced at OmniKassa.                                                                                                |
|                       | Otherwise, this field contains the time of the most recent payment attempt.                                                                                                                            |
| `errorCode`           | This field is reserved for future use. Currently, this field does not contain a value.                                                                                                                 |
| `paidAmount`          | The amount paid by the customer. In case of COMPLETED, it is equal to the order amount as given by the webshop at the announcement. When CANCELLED and EXPIRED, this field will have a value of 0.   |
| `totalAmount`         | The original order amount as specified by the webshop when announcing the order.                                                                                                                       |

<a name="request-available-payment-brands"></a>
### Request available payment brands

The SDK also provides functionality to look up all payment brands of a webshop as configured in the dashboard of Rabo OmniKassa.
This information can be used to determine which payment brands are available to the customer.
It is typically used in conjunction with the `paymentBrand` and `paymentBrandForce` fields of an order announcement ([Order announcement](#order-announcement)).

**Important:** The payment brands should not be requested real-time for each payment, but instead be cached locally and updated at certain time intervals (e.g. every few hours).

Given an `Endpoint` ([Creating endpoint](#creating-endpoint)) we can look up the payment brands as followings.

**PHP**
```php
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\PaymentBrandsResponse;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\PaymentBrandInfo;

$response = $endpoint->retrievePaymentBrands();
$paymentBrands = $response->getPaymentBrands();
foreach($paymentBrands as $paymentBrand) {
  $name = $paymentBrand->getName();
  $active = $paymentBrand->isActive();
  // use the name and the active variables for further processing
}
```


**Java**
```java
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.response.PaymentBrandsResponse;
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.response.PaymentBrandInfo;

PaymentBrandsResponse response = endpoint.retrievePaymentBrands();
for (PaymentBrandInfo paymentBrand : response.getPaymentBrands()) {
  String name = paymentBrand.getName();
  PaymentBrandStatus status = paymentBrand.getStatus();
  // use the name and status variables for further processing
}
```

**.NET**
```csharp
using OmniKassa.Model.Response;

PaymentBrandsResponse response = endpoint.RetrievePaymentBrands();
foreach(PaymentBrandInfo brand in response.PaymentBrands)
{
  String name = brand.Name;
  Boolean active = brand.IsActive;
  // use the name and active variables for further processing
}
```

As shown above the `retrievePaymentBrands()` method of the Endpoint class returns an instance of `PaymentBrandsResponse`.
This class provides a method `getPaymentBrands()` that returns a list containing all payment brands that are configured for the webshop.
Each element of this list is an object of type `PaymentBrandInfo` containing the details of a single payment brand, as shown in the following table.

| Field    | Description                                                                                                               |
|----------|---------------------------------------------------------------------------------------------------------------------------|
| `name`   | A string containing the name of the payment brand. This string may contain one of the following values:                   |
|          | IDEAL                                                                                                                     |
|          | PAYPAL                                                                                                                    |
|          | AFTERPAY                                                                                                                  |
|          | MASTERCARD                                                                                                                |
|          | VISA                                                                                                                      |
|          | BANCONTACT                                                                                                                |
|          | MAESTRO                                                                                                                   |
|          | V_PAY                                                                                                                     |
|          | SOFORT                                                                                                                    |
|          | Note: Keep in mind that other values can be expected as well whenever Rabo OmniKassa is extended with new payment brands. |
| `active` | A boolean indicating if the payment brand is active (true) or inactive (false).                                           |

Only the payment brands that are returned in this list and are active can be used for payments.

<a name="request-available-ideal-issuers"></a>
### Request available iDEAL issuers
In this section we explain how to obtain the iDEAL issuers. This functionality is typically used to directly start an
iDEAL transaction from the webshop without first redirecting the customer to the payment pages of Rabo OmniKassa to
select iDEAL as the payment brand and then the issuer.

**Important:** The list of iDEAL issuers should not be requested real-time for each payment, but instead be cached
locally and updated daily.

Given an `Endpoint` ([Creating endpoint](#creating-endpoint)) we can obtain this list as follows.

**PHP**
```php
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\response\IdealIssuersResponse;
use nl\rabobank\gict\payments_savings\omnikassa_sdk\model\response\IdealIssuersInfo;

$response = $endpoint->retrieveIDEALIssuers();
$issuers = $response->getIssuers();
foreach($issuer as $issuers) {
  // use the properties in the issuer variable for further processing
}
```

**Java**
```java
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.response.IdealIssuersResponse;
import nl.rabobank.gict.payments_savings.omnikassa_frontend.sdk.model.response.IdealIssuer;

IdealIssuersResponse response = endpoint.retrieveIdealIssuers();
for (IdealIssuer issuer : response.getIssuers()) {
  // use the properties in the issuer variable for further processing
}
```

**.NET**
```csharp
using OmniKassa.Model.Response;

IdealIssuersResponse response = endpoint.RetrieveIdealIssuers();
foreach(IdealIssuer issuer in response.IdealIssuers)
{
  // use the properties in the issuer variable for further processing
}
```

As shown above the `retrieveIdealIssers()` method of the Endpoint class returns an instance of `IdealIssuersResponse`.
This class provides a method `getIssuers()` that returns a list of all available iDEAL issuers.
Each element of this list is an object containing the details of a single iDEAL issuer as shown in the following table.

| Field          | Description                                                                                                                                                                                                                                                                      |
|--------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`           | A string of at most 11 characters containing the unique ID of the iDEAL issuer. This ID must be used in the `paymentBrandMetaData` field of an order to immediately start an iDEAL transaction for this issuer.                                                                  |
| `name`         | The name of the Issuer as should be presented to the customer when rendering the list of iDEAL issuers. This field is a string of at most 35 characters.                                                                                                                         |
| `logos`        | A list of objects containing logo details of the issuer. This information can be used to render the logo of the issuer in the webshop. See table below for more details on the logo properties.                                                                                  |
| `countryNames` | Contains the country names in the official languages of the country, separated by a '/' symbol. As prescribed by the iDEAL integration guide this only needs to be displayed if there are banks from more than one country on the Issuer list (which is currently not the case). |

The table below contains the properties of an iDEAL issuer logo.

| Field      | Description                                                                                |
|----------- | -------------------------------------------------------------------------------------------|
| `url`      | A publicly accessible URL where you can download the logo of the issuer.                   |
| `mimeType` | The mime type (also referred to as the media type) of the logo. This always an image type. |
