# Django Stripe Integration

This tutorial will teach you about how to integrate your django application with stripe.

### 1. Install & Configure Stripe

- Install stripe:

  ```shell
  pip install stripe
  ```

- Head to your stripe dashboard to get `api key` and `publishable key`: https://dashboard.stripe.com/test/developers

- Add stripe api key, publishable key to `settings.py` file:

  ```python
  STRIPE_SECRET_KEY = config('STRIPE_SECRET_KEY')
  STRIPE_PUBLISHABLE_KEY = config('STRIPE_PUBLISHABLE_KEY')
  ```

  - Note: The above code makes use of the `decouple` library to get environment variables from the .env file. In order to use this library, you have to first install it:

  ```shell
  pip install python-decouple
  ```

  And then import it at the top of your `settings.py` file:

  ```python
  from decouple import config
  ```

### 3. Stripe Checkout Endpoint

- Next, we need to setup the stripe checkout endpoint

- Import and initialize stripe in payment app `views.py` file:

  ```python
  from django.shortcuts import redirect
  from django.conf import settings
  from django.contrib.auth.decorators import login_required
  import stripe

  stripe.api_key = settings.STRIPE_SECRET_KEY
  ```

- In payment app `views.py`, create a new view to handle checkout:

  ```python
  @login_required
  def CreateCheckoutSessionView(request, product_id):
      ...
  ```

- In the view, we need to first get the producy using the product_id url parameter:

  ```python
  @login_required
  def CreateCheckoutSessionView(request, product_id):
      product = Product.objects.get(id=product_id)
  ```

- Next, create a stripe checkout session:

  - More documentation for this is here: https://docs.stripe.com/checkout/quickstart?client=html

  ```python
  @login_required
  def CreateCheckoutSessionView(request, product_id):
      product = Product.objects.get(id=product_id)

      YOUR_DOMAIN = f"{request.scheme}://{request.get_host()}"

      checkout_session = stripe.checkout.Session.create(
          payment_method_types=['card'],
          line_items=[
              {
                  'price_data': {
                      'currency': 'usd',
                      'unit_amount': product.price * 100,
                      'product_data': {
                          'name': product.name,
                          'images': [f'{product.image}']
                      },
                  },
                  'quantity': 1,
              },
          ],
          metadata = {
              'product_id': product_id,
              'user_email': request.user.email
          },

          mode='payment',

          success_url=YOUR_DOMAIN + f'/payment-success/{product_id}',
          cancel_url=YOUR_DOMAIN + f'/pricing/{product_id}',
      )

      return redirect(checkout_session.url)
  ```

- Create a url for checkout endpoint:

  ```python
  path('checkout/<int:product_id>/', views.CreateCheckoutSessionView, name='checkout'),
  ```

- Next, update the `pricing.html` file to properly redirect to the stripe check out page when the `purchase` button is clicked:

  ```html
  <!-- wrap button in anchor tag  -->
  <a href="{% url 'checkout' product.id %}"><button>Purchase</button></a>
  ```

- Test the integration with test card:
  ```shell
  4242424242424242
  ```

### 4. Get Stripe Webhook Secret

- What is a webhook?

  - ![My animated logo](webhook.png)

- Get webhook secret from stripe dashboard:
  ```shell
  https://dashboard.stripe.com/test/webhooks
  ```
- What is ngrok?

- Install ngrok:

  ```shell
  https://download.ngrok.com/
  ```

- Create a new ngrok url to expose local development server to public internet access:

  - Make sure your django development server is running and the run the following command:
    ```shell
    ngrok http 8000
    ```

- Copy the generated ngrok url and paste into the `Endpoint URL` field in stripe dashboard

- Optionally add a description about what this endpoint is for

- Slect the following events to listen to:

  ```shell
  checkout.session.completed
  checkout.session.async_payment_failed
  ```

- Go back to your `views.py` file and create a view to handle this webhook and map view to url:

  ```python
  from django.views.decorators.csrf import csrf_exempt

  def stripe_webhook(request):
      ...
  ```

  In your urls.py file:

  ```python
  path('stripe/webhook/', views.stripe_webhook, name='stripe-webhook'),
  ```

- Next, head back to your browser and copy the code provided by stripe towards the right-hand side of the page:

  ```python
  from django.views.decorators.csrf import csrf_exempt

  @csrf_exempt
  def stripe_webhook(request):
     event = None
     payload = request.data
     sig_header = request.headers['STRIPE_SIGNATURE']

     try:
         event = stripe.Webhook.construct_event(
             payload, sig_header, endpoint_secret
         )
     except ValueError as e:
         # Invalid payload
         raise e
     except stripe.error.SignatureVerificationError as e:
         # Invalid signature
         raise e

     # Handle the event
     if event['type'] == 'checkout.session.completed':
         session = event['data']['object']
     elif event['type'] == 'checkout.session.async_payment_failed':
         session = event['data']['object']

     else:
         print('Unhandled event type {}'.format(event['type']))

     return jsonify(success=True)
  ```

- Now we will make changes to this code because it is written for flask:

  - First is to change the `payload` and `sig_header` to this:
    ```python
    payload = request.body # replaced payload = request.data
    sig_header = request.META['HTTP_STRIPE_SIGNATURE'] # replaced request.headers['STRIPE_SIGNATURE']
    ```
  - Correctly pass in the `endpoint_secret`:
    ```python
    settings.STRIPE_WEBHOOK_SECRET # replaced endpoint_secret
    ```
  - Next, inside each exception, instead of raising an exception, we will return a HTTP response back to stripe:
    ```python
    return HttpResponse(status=400) # replaced raise e
    ```
  - Replace the final response with this:
    ```python
    return HttpResponse(status=200) # replaced return jsonify(success=True)
    ```

- Go back to browser and click `Add events` and the click `Add endpoint`

- Copy and Save webhook secret in `settings.py` file:
  ```python
  STRIPE_WEBHOOK_SECRET = config('STRIPE_WEBHOOK_SECRET')
  ```

### 5. Handle Successful and Failed Events

- For successful events, we can proceed to create a new purchase `PurchaseHistory` for this product and optionally email the user.

  - We can get the product id and user email below:

    ```python
    if event['type'] == 'checkout.session.completed':
        session = event['data']['object']

        product_id = session['metadata']['product_id']
        user_email = session['metadata']['user_email']
    ```

  - Next, we can create a new `PurchaseHistory`:

    ```python
    if event['type'] == 'checkout.session.completed':
       session = event['data']['object']

       product_id = session['metadata']['product_id']
       user_email = session['metadata']['user_email']

       get_product = Product.objects.get(id=product_id)

       PurchaseHistory.objects.create(product=get_product, purchase_success=True)

       # send success email to user to let them know purchase is complete
       ...
    ```

  - Finally, we can handle a failed purchase here:

    ```python
    elif event['type'] == 'checkout.session.async_payment_failed':
        session = event['data']['object']

        product_id = session['metadata']['product_id']
        user_email = session['metadata']['user_email']

        get_product = Product.objects.get(id=product_id)

        PurchaseHistory.objects.create(product=get_product)

        # send failed email to user to let them know purchase is complete
    ```

### 6. Test the code

- Generate a new ngrok url for local development server and replace old url in webhook setting in stripe dashboard:

  ```shell
  ngrok http 8000
  ```

## Question:

`Stripe` provides a testing environment for developers. We can get `Publishable` key and `Secret` key there. I think in real world projects, we get Publishable keys and Secret keys too. My question is, what other features we can test in this environment?

## Answer:

Stripe's test environment is much more than just a place to use fake keys; it is a full-scale simulation of the global financial system. Professional developers use it to "break" their app in every possible way before a real customer ever sees it.

Here are the most powerful features you can test in this environment:

---

### 1. Specific Payment Failures

You don't just test "success." Stripe provides a list of **magic card numbers** that trigger specific bank responses. This allows you to test if your Django code handles errors gracefully.

- **Insufficient Funds:** Use a specific card number to see how your UI tells the user they need more money.
- **Card Declined:** Test how your app handles a generic "no" from the bank.
- **Fraud Triggers:** Use a "stolen" card number to verify that Stripe Radar blocks it and that your app doesn't accidentally fulfill the order.

### 2. 3D Secure (SMS Verification)

You can simulate the "SMS code" popup. Stripe provides test cards that **force** a 3D Secure challenge. This is vital for testing how your frontend handles the transition from your site to the bank's verification screen and back.

### 3. Test Clocks (For Subscriptions)

If you were building a subscription service (e.g., $10/month), you wouldn't want to wait 30 days to see if your "Renewal" logic works.

- **Time Travel:** Stripe has a feature called **Test Clocks**. You can create a customer, attach them to a clock, and then "fast-forward" that clock by 1 month.
- **The Result:** Stripe will instantly trigger the next month's invoice and webhook as if a month has actually passed.

### 4. Webhook Triggering

You don't have to actually complete a payment to test your webhook code. You can use the **Stripe CLI** to manually "fire" any event you want.

```bash
# This sends a 'payment succeeded' event to your local Django server
stripe trigger payment_intent.succeeded

```

This is how professionals ensure their backend logic (like sending confirmation emails) works without needing to fill out a checkout form every time.

### 5. Disputes and Refunds

You can simulate a customer filing a **Dispute** (chargeback).

- In the dashboard, you can "win" or "lose" the dispute to see how your backend reacts to the money being pulled out of your account.
- You can also test **Refunds** to ensure that if you refund a user, your `Order` status in Django updates from `paid` back to `refunded`.

---

### Summary Table: Test Mode vs. Live Mode

| Feature      | Test Mode                  | Live Mode                      |
| ------------ | -------------------------- | ------------------------------ |
| **Money**    | Simulated ($0.00 cost)     | Real (Actual charges)          |
| **Cards**    | Only "Magic" numbers work  | Real Visa/Mastercards only     |
| **Emails**   | Sent only to you (The Dev) | Sent to the real Customer      |
| **Disputes** | Simulated outcomes         | Real legal process             |
| **Webhooks** | Can be manually triggered  | Triggered by real world events |

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:
