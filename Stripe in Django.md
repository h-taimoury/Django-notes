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
