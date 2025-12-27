# Stripe Integration Guide for AutoMSP

## Overview
Stripe is integrated into AutoMSP to handle all payment processing, subscription management, and billing operations. This guide provides comprehensive documentation for setting up and managing Stripe integration.

## Setup Instructions

### 1. Create Stripe Account
1. Visit https://dashboard.stripe.com/register
2. Sign up with business email
3. Verify email address
4. Complete account setup with business information
5. Enable 2FA for security

### 2. Obtain API Keys
1. Navigate to Developers > API Keys
2. Copy your publishable and secret keys
3. Store secret keys securely in environment variables
4. Never commit API keys to version control

### 3. Environment Variables
```bash
STRIPE_PUBLIC_KEY=pk_live_your_public_key
STRIPE_SECRET_KEY=sk_live_your_secret_key
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret
```

## Implementation

### Payment Processing

#### Subscription Creation
```javascript
const subscription = await stripe.subscriptions.create({
  customer: customerId,
  items: [
    { price: 'price_tier_professional' }
  ],
  metadata: {
    mspId: mspId,
    tier: 'professional'
  },
  billing_cycle_anchor: billingDate
});
```

#### One-Time Payments
```javascript
const paymentIntent = await stripe.paymentIntents.create({
  amount: 5000, // $50.00
  currency: 'usd',
  payment_method_types: ['card'],
  metadata: {
    mspId: mspId,
    service: 'implementation'
  }
});
```

### Webhook Configuration

#### Webhook Endpoint Setup
1. Go to Developers > Webhooks
2. Add endpoint: `https://api.automsp.com/webhooks/stripe`
3. Select events to listen:
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.payment_succeeded`
   - `invoice.payment_failed`
   - `charge.refunded`

#### Webhook Handler
```javascript
app.post('/webhooks/stripe', express.raw({type: 'application/json'}), async (req, res) => {
  const sig = req.headers['stripe-signature'];
  let event;

  try {
    event = stripe.webhooks.constructEvent(
      req.body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  switch(event.type) {
    case 'customer.subscription.created':
      await handleSubscriptionCreated(event.data.object);
      break;
    case 'invoice.payment_succeeded':
      await handlePaymentSucceeded(event.data.object);
      break;
    case 'invoice.payment_failed':
      await handlePaymentFailed(event.data.object);
      break;
  }

  res.json({received: true});
});
```

## Billing Features

### Subscription Management
- Automatic recurring billing on defined intervals
- Support for proration when upgrading/downgrading
- Billing cycle customization
- Invoice generation and delivery
- Retry logic for failed payments

### Pricing Models

#### Usage-Based Billing
```javascript
const subscription = await stripe.subscriptions.create({
  customer: customerId,
  items: [
    {
      price: 'price_usage_based',
      billing_thresholds: {
        usage_gte: 100
      }
    }
  ]
});
```

#### Tiered Pricing
```javascript
const price = await stripe.prices.create({
  product: productId,
  unit_amount: 2999, // $29.99
  currency: 'usd',
  recurring: {
    interval: 'month'
  },
  tiers: [
    { up_to: 100, unit_amount_decimal: '2999' },
    { up_to: 500, unit_amount_decimal: '2499' },
    { up_to_infinity: true, unit_amount_decimal: '1999' }
  ]
});
```

## Security Best Practices

### PCI Compliance
- Never handle raw card data server-side
- Always use Stripe's hosted payment forms
- Use Stripe Elements or Checkout
- Maintain PCI DSS compliance

### Data Protection
- Encrypt all sensitive data at rest
- Use HTTPS for all connections
- Implement rate limiting on API endpoints
- Monitor for suspicious activity
- Regular security audits

### API Key Management
- Rotate keys quarterly
- Use restricted API keys for specific operations
- Implement key rotation policies
- Monitor key usage logs
- Disable unused keys immediately

## Testing

### Test Card Numbers
- Visa: 4242 4242 4242 4242
- Mastercard: 5555 5555 5555 4444
- Amex: 3782 822463 10005
- Declined: 4000 0000 0000 0002

### Test Mode
1. Use publishable key: `pk_test_...`
2. Use secret key: `sk_test_...`
3. All charges are simulated
4. Switch to live mode for production

## Monitoring & Reporting

### Key Metrics
- Monthly Recurring Revenue (MRR)
- Churn rate
- Customer Lifetime Value (LTV)
- Failed payment rate
- Revenue per customer

### Stripe Dashboard Analytics
1. Revenue by product
2. Subscription analytics
3. Failed payments
4. Customer retention metrics
5. Geographic distribution

## Troubleshooting

### Common Issues

#### Failed Payments
- Check card validity
- Verify customer has valid payment method
- Check for insufficient funds
- Review declined transaction codes

#### Webhook Delivery Issues
- Verify endpoint is accessible
- Check webhook secret configuration
- Review webhook event logs
- Implement retry logic
- Monitor event processing delays

#### Integration Problems
- Verify API keys are correct
- Check for network connectivity
- Review API rate limits
- Check Stripe system status
- Enable debug logging

## Migration Guide

### Migrating from Other Payment Processors
1. Set up Stripe account and API keys
2. Migrate customer data:
   - Create Stripe customers
   - Tokenize payment methods
   - Set up subscription metadata
3. Test thoroughly with sample customers
4. Run parallel billing for transition period
5. Cutover to Stripe billing
6. Monitor for issues post-migration

## Support & Resources

- **Documentation:** https://stripe.com/docs
- **API Reference:** https://stripe.com/docs/api
- **Support:** https://support.stripe.com
- **Status Page:** https://status.stripe.com
- **Community:** https://stripe.com/community

## Compliance & Legal

### Terms of Service
- Review Stripe Services Agreement
- Comply with Stripe's Acceptable Use Policy
- Follow data protection regulations
- Maintain PCI DSS compliance

### Data Privacy
- GDPR compliance for European customers
- CCPA compliance for California residents
- Regular data retention audits
- Implement customer data deletion requests

## Next Steps

1. Complete Stripe account setup
2. Obtain and configure API keys
3. Set up webhook endpoints
4. Implement payment processing
5. Test in sandbox environment
6. Deploy to production
7. Monitor and optimize performance
