Cashier Register - Track the plan quotas
===========================================

![CI](https://github.com/renoki-co/cashier-register/workflows/CI/badge.svg?branch=master)
[![codecov](https://codecov.io/gh/renoki-co/cashier-register/branch/master/graph/badge.svg)](https://codecov.io/gh/renoki-co/cashier-register/branch/master)
[![StyleCI](https://github.styleci.io/repos/277109456/shield?branch=master)](https://github.styleci.io/repos/277109456)
[![Latest Stable Version](https://poser.pugx.org/renoki-co/cashier-register/v/stable)](https://packagist.org/packages/renoki-co/cashier-register)
[![Total Downloads](https://poser.pugx.org/renoki-co/cashier-register/downloads)](https://packagist.org/packages/renoki-co/cashier-register)
[![Monthly Downloads](https://poser.pugx.org/renoki-co/cashier-register/d/monthly)](https://packagist.org/packages/renoki-co/cashier-register)
[![License](https://poser.pugx.org/renoki-co/cashier-register/license)](https://packagist.org/packages/renoki-co/cashier-register)

Cashier Register is a simple quota feature usage tracker for Laravel Cashier subscriptions.

It helps you define static, project-level plans, and attach them features that can be tracked and limited throughout the app. For example, you might want to set a limit of `5` seats per team and make it so internally. Cashier Register comes with a nice wrapper for Laravel Cashier that does that out-of-the-box.

- [Cashier Register - Track the plan quotas](#cashier-register---track-the-plan-quotas)
  - [🤝 Supporting](#-supporting)
  - [🚀 Installation](#-installation)
  - [🙌 Usage](#-usage)
  - [Preparing the model](#preparing-the-model)
  - [Preparing the plans](#preparing-the-plans)
    - [Feature Usage Tracking](#feature-usage-tracking)
    - [Checking for overflow](#checking-for-overflow)
    - [Metered billing when overflowing](#metered-billing-when-overflowing)
    - [Resetting tracked values](#resetting-tracked-values)
    - [Unlimited amounts](#unlimited-amounts)
    - [Checking for overexceeded quotas](#checking-for-overexceeded-quotas)
    - [Metered features](#metered-features)
    - [Additional data](#additional-data)
    - [Setting the plan as popular](#setting-the-plan-as-popular)
    - [Inherit features from other plans](#inherit-features-from-other-plans)
  - [Static items](#static-items)
  - [🐛 Testing](#-testing)
  - [🤝 Contributing](#-contributing)
  - [🔒  Security](#--security)
  - [🎉 Credits](#-credits)

## 🤝 Supporting

Renoki Co. on GitHub aims on bringing a lot of open source projects and helpful projects to the world. Developing and maintaining projects everyday is a harsh work and tho, we love it.

If you are using your application in your day-to-day job, on presentation demos, hobby projects or even school projects, spread some kind words about our work or sponsor our work. Kind words will touch our chakras and vibe, while the sponsorships will keep the open source projects alive.

[![ko-fi](https://www.ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/R6R42U8CL)

## 🚀 Installation

You can install the package via composer:

```bash
composer require renoki-co/cashier-register
```

The package does not come with Cashier as dependency, so you should install according to your needs:

```
$ composer require laravel/cashier:"^12.13"
```

For Paddle, use Cashier for Paddle:

```
$ composer require laravel/cashier-paddle:"^1.4.4"
```

Publish the config file:

```bash
$ php artisan vendor:publish --provider="RenokiCo\CashierRegister\CashierRegisterServiceProvider" --tag="config"
```

Publish the migrations:

```bash
$ php artisan vendor:publish --provider="RenokiCo\CashierRegister\CashierRegisterServiceProvider" --tag="migrations"
```

## 🙌 Usage

``` php
use RenokiCo\CashierRegister\CashierRegisterServiceProvider as BaseServiceProvider;
use RenokiCo\CashierRegister\Saas;

class CashierRegisterServiceProvider extends BaseServiceProvider
{
    /**
     * Boot the service provider.
     *
     * @return void
     */
    public function boot()
    {
        parent::boot();

        Saas::plan('Gold Plan', 'monthly-price-id', 'yearly-price-id')
            ->description('The gold plan.')
            ->monthly(30)
            ->yearly(300)
            ->currency('EUR')
            ->features([
                Saas::feature('Build Minutes', 'build.minutes', 3000)
                    ->description('3000 build minutes for an entire month!'),
            ]);
    }
}
```

```php
$user->subscription('main')
    ->recordFeatureUsage('build.minutes', 30);
```

**Please note: The Yearly Price ID and the Yearly Price are optional.**

## Preparing the model

For billables, you should follow the installation instructions given with Cashier for Paddle or Cashier for Stripe.

This package already sets the custom `Subscription` model. In case you want to add more functionalities to the Subscription model, make sure you extend accordingly from these models:

- Paddle: `RenokiCo\CashierRegister\Models\Paddle\Subscription`
- Stripe: `RenokiCo\CashierRegister\Models\Stripe\Subscription`

Further, make sure you check the `saas.php` file and replace the subscription model from there, or you can use the `::useSubscriptionModel` call in your code.

## Preparing the plans

You can define the plans at the app service provider level and it will stick throughout the request cycle.

First of all, publish the Provider file:

```bash
$ php artisan vendor:publish --provider="RenokiCo\CashierRegister\CashierRegisterServiceProvider" --tag="provider"
```

Import the created `app/Providers/CashierRegisterServiceProvider` class into your `app.php`:

```php
$providers = [
    // ...
    \App\Providers\CashierRegisterServiceProvider::class,
];
```

In `CashierRegisterServiceProvider`'s `boot` method you may define the plans you need:

```php
use RenokiCo\CashierRegister\CashierRegisterServiceProvider as BaseServiceProvider;
use RenokiCo\CashierRegister\Saas;

class CashierRegisterServiceProvider extends BaseServiceProvider
{
    /**
     * Boot the service provider.
     *
     * @return void
     */
    public function boot()
    {
        parent::boot();

        // Define plans here.
    }
}
```

**When setting an unique identifier for the plan (second parameter), make sure to use the Stripe Price ID or the Paddle Plan ID.**

Defining plans can also help you retrieving them when showing them in the frontend:

```php
use RenokiCo\CashierRegister\Saas;

$allPlans = Saas::getPlans();

foreach ($allPlans as $plan) {
    $features = $plan->getFeatures();

    //
}
```

Or retrieving a specific plan by Plan ID:

```php
use RenokiCo\CashierRegister\Saas;

$plan = Saas::getPlan('plan-id');
```

Deprecating plans can occur anytime. In order to do so, just call `deprecated()` when defining the plan:

```php
/**
 * Boot the service provider.
 *
 * @return void
 */
public function boot()
{
    parent::boot();

    Saas::plan('Silver Plan', 'silver-plan-id')->deprecated();
}
```

As an alternative, you can anytime retrieve the available plans only:

```php
use RenokiCo\CashierRegister\Saas;

$plans = Saas::getAvailablePlans();
```

### Feature Usage Tracking

You can attach features to the plans:

```php
use RenokiCo\CashierRegister\CashierRegisterServiceProvider as BaseServiceProvider;
use RenokiCo\CashierRegister\Saas;

class CashierRegisterServiceProvider extends BaseServiceProvider
{
    /**
     * Boot the service provider.
     *
     * @return void
     */
    public function boot()
    {
        parent::boot();

        Saas::plan('Gold Plan', 'gold-plan')->features([
            Saas::feature('Build Minutes', 'build.minutes', 3000)
                ->description('3000 build minutes for an entire month!'),
        ]);
    }
}
```

Then track them:

```php
$subscription->recordFeatureUsage('build.minutes', 30); // reducing 30 mins

$subscription->getUsedQuota('build.minutes') // 30
$subscription->getRemainingQuota('build.minutes') // 2950
```

### Checking for overflow

Checking overflow can be useful when users fallback from a bigger plan to an older plan. In this case, you may end up with an overflow case where the users will have feature tracking values greater than the smaller plan values.

You can check if the feature value overflown by calling `featureOverQuota`:

```php
$subscription->swap($freePlan); // has no build minutes

// Will return true if the consumed build minutes are greater than the free plan (0 minutes)
$subscription->featureOverQuota('build.minutes');
```

Naturally, `recordFeatureUsage()` has a callback method that gets called whenever the amount of consumption gets over the allocated total quota.

For example, users can have 1000 build minutes each month, but at some point if they have 10 left and they consume 15, the feature usage will be saturated/depleted completely, and the extra amount will be passed to the callback:

```php
$subscription->recordFeatureUsage('build.minutes', 15, true, function ($feature, $valueOverQuota, $subscription) {
    // Bill the user with on-demand pricing, per se.
    $this->billOnDemandFor($valueOverQuota, $subscription);
});
```

### Metered billing when overflowing

When exceeding the allocated quota for a specific feature, [Metered Billing for Stripe](#metered-features) can come in and bill for metered usage, but only if it's a Metered Feature and the quota is exceeded and the feature is defined as [Metered Feature](#metered-features).

```php
Saas::plan('Gold Plan', 'gold-plan')->features([
    Saas::meteredFeature('Build Minutes', 'build.minutes', 3000), // included: 3000
        ->meteredPrice('price_identifier', 0.01, 'minute'), // on-demand: $0.01/minute
]);

$subscription->recordFeatureUsage('build.minutes', 4000, true, function ($feature, $valueOverQuota, $subscription) {
    // From the used 4000 minutes, 3000 were included already by the plan feature.
    // Extra 1000 (to reach a total usage of 4000) is over the quota, so because
    // the feature is metered and we defined a ->meteredPrice(), the package
    // wil automatically record the usage to Stripe via Cashier.

    // Here you can run custom logic to handle overflow.
});
```

### Resetting tracked values

By default, each created feature is resettable - each time the billing cycle ends, you can call `resetQuotas` to reset them (they will become 3000 in the previous example).

Make sure to call `resetQuotas` after the billing cycle resets.

For example, you can extend the default Stripe Webhook controller that Laravel Cashier comes with and implement the `invoice.payment_succeeded` event handler:

```php
<?php
use Laravel\Cashier\Http\Controllers\WebhookController;

class StripeController extends WebhookController
{
    /**
     * Handle invoice payment succeeded.
     *
     * @param  array  $payload
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function handleInvoicePaymentSucceeded($payload)
    {
        if ($user = $this->getUserByStripeId($payload['data']['object']['customer'])) {
            $data = $payload['data']['object'];

            $subscription = $user->subscriptions()
                ->whereStripeId($data['subscription'] ?? null)
                ->first();

            if ($subscription) {
                $subscription->resetQuotas();
            }
        }

        return $this->successMethod();
    }
}
```

To avoid resetting, like counting the seats for a subscription, you should call `notResettable()` on the feature:

```php
Saas::plan('Gold Plan', 'gold-plan')->features([
    Saas::feature('Seats', 'seats', 5)->notResettable(),
]);
```

Now when calling `resetQuotas()`, the `seats` feature won't go back to the default value.

### Unlimited amounts

To set an infinite amount of usage, use the `unlimited()` method:

```php
Saas::plan('Gold Plan', 'gold-plan')->features([
    Saas::feature('Seats', 'seats')->unlimited(),
]);
```

### Checking for overexceeded quotas

When swapping from a bigger plan to a small plan, you might restrict users from doing it unless all the quotas do not exceed the smaller plan's quotas.

For example, an user can subscribe to a plan that has 10 teams, it creates 10 teams, but later decides to downgrade. In this case, you can check which features
are exceeding:

```php
$freePlan = Saas::plan('Free Plan', 'free-plan');
$paidPlan = Saas::plan('Paid Plan', 'paid-plan');

// Returning an Illuminate\Support\Collection instance with each
// item as RenokiCo\CashierRegister\Feature instance.
$overQuotaFeatures = $subscription->featuresOverQuotaWhenSwapping(
    $paidPlan->getId()
);

foreach ($overQuotaFeatures as $feature) {
    // $feature->getName();
}
```

**Please keep in mind that this works only for non-resettable features, like Teams, Members, etc. due to the fact that features, when swapping between plans, should be handled manually, either wanting to keep them as-is or resetting them using `resetQuotas()`**

### Metered features

Metered features are opened for Stripe only and this will open up custom metering for exceeding quotas on features.

You might want to give your customers a specific amount of a feature, let's say `Build Minutes`, but for exceeding amount of minutes you might invoice at the end of the month a price of `$0.01` per minute:

```php
Saas::plan('Gold Plan', 'gold-plan')->features([
    Saas::meteredFeature('Build Minutes', 'build.minutes', 3000), // included: 3000
        ->meteredPrice('price_identifier', 0.01, 'minute'), // on-demand: $0.01/minute
]);
```

If you simply want just the on-demand price of the metered feature, just omit the amount:

```php
Saas::plan('Gold Plan', 'gold-plan')->features([
    Saas::meteredFeature('Build Minutes', 'build.minutes'), // included: 0
        ->meteredPrice('price_identifier', 0.01, 'minute'), // on-demand: $0.01/minute
]);
```

**The third parameter is just a conventional name for the unit. `0.01` is the price per unit (PPU). In this case, it's `minute` and `$0.01`, assuming the plan's price is in USD.**

### Additional data

Items, plans and features implement a `->data()` method that allows you to attach custom data for each item:

```php
Saas::plan('Gold Plan', 'gold-plan')
    ->data(['golden' => true])
    ->features([
        Saas::feature('Seats', 'seats')
            ->data(['digital' => true])
            ->unlimited(),
    ]);

$plan = Saas::getPlan('gold-plan');
$feature = $plan->getFeature('seats');

$planData = $plan->getData(); // ['golden' => true]
$featureData = $feature->getData(); // ['digital' => true]
```

### Setting the plan as popular

Some plans are popular among others, and you can simply mark them:

```php
Saas::plan('Gold Plan', 'gold-plan')
    ->popular();
```

This will add a data field called `popular` that is either `true/false`.

### Inherit features from other plans

You may copy the base features from a given plan and overwrite same-ID features for new plans.

```php
$freePlan = Saas::plan('Free Plan', 'free-plan')->features([
    Saas::feature('Seats', 'seats')->value(10),
]);

$paidPlan = Saas::plan('Paid Plan', 'paid-plan')->inheritFeaturesFromPlan($freePlan, [
    Saas::feature('Seats', 'seats')->unlimited(), // same-ID features are replaced
    Saas::feature('Beta Access', 'beta.access')->unlimited(), // new IDs are merged
]);
```

The second argument passed to the function is the array of features to replace within the current Free Plan.

**Keep in mind, avoid using further `->features()` when inheriting from another plan.**

## Static items

In case you are not using plans, you can describe items once in Cashier Register's service provider and then leverage it for some neat usage:

```php
Saas::item('Elephant Sticker', 'elephant-sticker')
    ->price(5, 'EUR');
```

Then later be able to retrieve it:

```php
$item = Saas::getItem('elephant-sticker');

$item->getPrice(); // 5
$item->getCurrency(); // 'EUR'
```

Each item can have sub-items too:

```php
Saas::item('Sticker Pack', 'sticker-pack')
    ->price(20, 'EUR')
    ->subitems([
        Saas::item('Elephant Sticker', 'elephant-sticker')->price(5, 'EUR'),
        Saas::item('Zebra Sticker', 'zebra-sticker')->price(10, 'EUR'),
    ]);

$item = Saas::getItem('sticker-pack');

foreach ($item->getSubitems() as $item) {
    $item->getName(); // Elephant Sticker, Zebra Sticker, etc...
}
```

## 🐛 Testing

``` bash
vendor/bin/phpunit
```

## 🤝 Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## 🔒  Security

If you discover any security related issues, please email alex@renoki.org instead of using the issue tracker.

## 🎉 Credits

- [Alex Renoki](https://github.com/rennokki)
- [All Contributors](../../contributors)
