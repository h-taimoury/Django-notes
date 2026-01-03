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

## Q&A Section

## Note:

Visit this [link](https://docs.stripe.com/api/) to read the documentation about Stripe API. Specifically use this [link](https://docs.stripe.com/api/checkout/sessions/create) to read documentation for `stripe.checkout.Session.create` method.

## Question:

Tell me what is `Stripe`? What other options do we have other than using Stripe for an E-Commerce platform? What services does Stripe provide?

## Answer:

**Stripe** is essentially the "financial internet." For a developer, it's a massive collection of APIs that allow you to move money around the world without needing to build your own bank or security infrastructure.

### 1. What exactly is Stripe?

Stripe is a **full-stack payment service provider**. In the old days of the web, you needed two separate things to sell online:

1. **A Payment Gateway:** The secure "form" where customers type their card info.
2. **A Merchant Account:** A special bank account that can hold the money before it's sent to your personal bank.

Stripe combines both. They provide the gateway (the software) and the merchant account (the financial pipe).

---

### 2. What other options are there?

Depending on your business model, you might choose a competitor. Here is how they stack up in 2025:

| Competitor                 | Best For...            | Why Choose Them Over Stripe?                                                                                                        |
| -------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **PayPal**                 | **Trust & Conversion** | Millions of users already have a PayPal account and trust their "Buyer Protection."                                                 |
| **Square**                 | **Omnichannel**        | If you have a physical coffee shop _and_ an online store, Square syncs your inventory and payments perfectly.                       |
| **Adyen**                  | **Enterprise**         | Used by giants like Uber and Netflix. They have better rates for companies doing billions in sales.                                 |
| **Paddle / Lemon Squeezy** | **Digital Goods/SaaS** | They act as a **Merchant of Record**. They handle all the messy global sales tax (VAT) for you, which Stripe doesn't do by default. |
| **Braintree**              | **Customization**      | Owned by PayPal, but offers a cleaner API similar to Stripe for enterprise-level customization.                                     |

---

### 3. What services does Stripe provide?

Stripe has expanded far beyond just "Credit Card Processing." They now offer an entire suite of financial tools:

- **Stripe Checkout / Elements:** The frontend forms you are currently using.
- **Stripe Connect:** Allows you to build a **Marketplace** (like Etsy or Uber). You can take a payment, keep a 10% fee, and automatically send the remaining 90% to a "seller."
- **Stripe Billing:** Specialized logic for **Subscriptions**. It handles recurring payments, "free trials," and sending emails when a card expires.
- **Stripe Radar:** An AI-powered fraud detector that automatically blocks "shady" looking transactions.
- **Stripe Atlas:** A service that helps founders legally incorporate a business in the US (Delaware) from anywhere in the world.
- **Stripe Tax:** Automatically calculates how much sales tax or VAT you need to charge based on the customer's address.
- **Stripe Issuing:** Allows you to create your own **physical or virtual credit cards** for your employees or customers.

---

### Why we chose Stripe for your project

As a frontend developer, you likely noticed that Stripe's documentation is arguably the best in the industry. They prioritize the **Developer Experience (DX)**. Most competitors have older, more confusing APIs, while Stripe feels like it was "built by developers, for developers."

## Question:

Walk me through the workflow from frontend to backend and `Stripe`. When a user clicks `Checkout` button, on the frontend we'll navigate him to checkout page where he can see the order's details and a `Pay` buttton. When he clicks this button, what happens next? Tell me exactly what requests are made to backend or Stripe and which API endpoints get involved. I don't want you to tell me very specific details about the requests, just a step by step overview.

## Answer:

When a user clicks that final "Pay" button on your site, they initiate a high-speed game of "digital telephone" between your Frontend, your Django Backend, and Stripe's servers.

Here is the exact step-by-step workflow:

### Step 1: Frontend to Backend (The Request)

- **Action:** The user clicks the "Pay" button.
- **Request:** Your **Frontend** sends a `POST` request to your **Django Backend** (e.g., `/api/payments/create-session/`).
- **Data Sent:** Usually just the `order_id`.

### Step 2: Backend to Stripe (The Handshake)

- **Action:** Your Django view receives the `order_id`. It fetches the order details (price, items) from your database.
- **Request:** Your **Backend** sends a `POST` request to the **Stripe API** (`/v1/checkout/sessions`).
- **Data Sent:** Your Secret Key, the price, the success/cancel URLs, and a `client_reference_id` (your `order_key`).
- **Response:** Stripe responds with a **Session Object** which contains a unique, temporary **URL**.

### Step 3: Backend to Frontend (The "Go" Signal)

- **Action:** Your **Backend** sends a `JSON` response back to your **Frontend**.
- **Data Sent:** The Stripe `checkout_url`.

### Step 4: Redirection (The User Leaves Your Site)

- **Action:** Your **Frontend** receives the URL and immediately redirects the user's browser.
- **Destination:** The user is now on `checkout.stripe.com`. They see your logo and the payment form.

### Step 5: Stripe to User's Bank (The Money Move)

- **Action:** The user enters their card info and clicks "Pay" on the Stripe page.
- **Communication:** **Stripe** communicates directly with the **Bank** (and potentially triggers an SMS verification/3DS).
- **Result:** The bank approves or declines.

### Step 6: Stripe to Backend (The Webhook - Critical!)

- **Action:** As soon as the payment is successful, **Stripe** sends a `POST` request to your **Backend Webhook** (`/api/payments/webhook/`).
- **Why?** This happens independently of the user's browser. Even if the user closes their laptop, this request still arrives.
- **Backend Logic:** Your Django app verifies the signature, looks up the `Order` using the ID Stripe sent back, and changes its status from `pending` to `paid`.

### Step 7: Final Redirection (The Homecoming)

- **Action:** **Stripe** redirects the user's browser back to your site's `success_url`.
- **Result:** The user sees a "Thank you for your purchase!" page.

---

### Summary of Endpoint Roles

| Player           | Endpoint                        | Role                                                   |
| ---------------- | ------------------------------- | ------------------------------------------------------ |
| **Your Backend** | `/api/payments/create-session/` | Validates the order and asks Stripe for a session.     |
| **Stripe API**   | `/v1/checkout/sessions`         | Generates the hosted payment page.                     |
| **Your Backend** | `/api/payments/webhook/`        | Receives the final "Success" confirmation from Stripe. |

## Question:

What is a `Webhook`?

## Answer:

Here is the breakdown of what a Webhook is, based on your five points:

### 1. The "Hook" in Programming

In the programming world, a **"Hook"** is a mechanism that allows you to "hook into" a specific moment in a process to execute your own code.

- Imagine a software's internal workflow as a straight line.
- A developer leaves a "hook" (an empty slot) at a specific event (like "User Deleted").
- When that event happens, the software "catches" that hook and runs whatever custom code you’ve attached to it.

### 2. Why "Web" hook?

It is called a **Web**-hook because it uses the standard infrastructure of the World Wide Web to do the "hooking." Instead of using a physical cable or a complex internal system connection, Software A communicates with Software B using **HTTP (Hypertext Transfer Protocol)**.

- Stripe (Software A) is basically acting like a browser; it makes an HTTP POST request to your URL just like a user might submit a form.

### 3. The "Reverse API"

In a standard API scenario, you (the client) call the server to get data. In a Webhook scenario, the roles are reversed: **The server calls you.**

- This is why it’s called a **Reverse API**.
- Instead of your Django app saying, "Hey Stripe, did anyone pay yet?" (Polling), Stripe says, "Hey Django, here is the money!" (Pushing).

---

### 4. API vs. Webhook: The Difference

The fundamental difference is **who initiates the conversation**.

| Feature       | Standard API                                                       | Webhook                                                                             |
| ------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| **Direction** | **Pull:** You ask the server for data.                             | **Push:** The server sends data to you.                                             |
| **Timing**    | On your command (whenever you want).                               | On an event (only when something happens).                                          |
| **Analogy**   | **Vending Machine:** You go to it, press a button, and get a soda. | **Pizza Delivery:** You stay home; the doorbell rings only when the pizza is ready. |

---

### 5. The Ambiguity of the Term

The word "Webhook" is technically a description of the **connection mechanism**, but in daily developer talk, the meaning shifts based on the context:

- **The Sender:** Someone might say, "Stripe is our webhook," meaning Stripe is the service providing the event triggers.
- **The Message (The Payload):** You’ll often hear, "Did we receive the webhook yet?" In this context, they mean the **JSON data packet** that was sent after the event.
- **The Receiver (The Endpoint):** You might say, "I need to write a webhook in Django." Here, you mean the **URL/View** that is sitting and waiting to receive the data.

---

### Summary

A Webhook is an **HTTP-based callback** that allows two systems to be "hooked" together so that Software A can automatically notify Software B the very millisecond an event occurs.

## Question:

In this code:

```python
class PaymentService:
    @staticmethod
    def create_checkout_session(order: Order):
        # The method logic
```

What is this `@staticmethod` decorator?

## Answer:

The `@staticmethod` decorator is a built-in Python tool used to define a method inside a class that **does not need access to the class itself or its instances.**

Think of it as a "regular function" that just happens to live inside a class because it's logically related to it.

### 1. The Main Differences

In a Python class, you typically have three types of methods. The decorator you use tells Python what information to "automatically" pass to that function.

| Method Type         | Decorator           | First Argument | Purpose                                                                    |
| ------------------- | ------------------- | -------------- | -------------------------------------------------------------------------- |
| **Instance Method** | _None_              | `self`         | Can read/modify data specific to a single object (e.g., `user.name`).      |
| **Class Method**    | `@classmethod`      | `cls`          | Can read/modify data shared by the whole class (e.g., `User.total_count`). |
| **Static Method**   | **`@staticmethod`** | **None**       | Doesn't need to know about the class or the object. It's just a helper.    |

---

### 2. Why use it in your `PaymentService`?

In the code we wrote, we used `@staticmethod` for `create_checkout_session` because:

- **No State Needed:** The function doesn't need to check or change any "internal variables" of the `PaymentService` class. It just takes an `Order` and returns a URL.
- **Convenience:** You can call it directly using the class name without creating an object first:

```python
# NO: service = PaymentService()
#     url = service.create_checkout_session(order)

# YES: Call it directly like a library
url = PaymentService.create_checkout_session(order)

```

- **Organization:** It keeps your code tidy. Instead of having a "loose" function floating around in your file, you group all your payment-related logic under the `PaymentService` "namespace."

---

### 3. A Simple Comparison

If we didn't use `@staticmethod`, your code would look slightly more cluttered:

```python
class PaymentService:
    # Standard Instance Method (Needs 'self')
    def process_something(self, data):
        print(f"Working on {self}")

    @staticmethod
    def create_checkout_session(order):
        # I don't need 'self'! I just need the 'order'.
        return "https://stripe.com/..."

```

### Summary

When you see `@staticmethod`, it's a signal to other developers saying: **"This function is a pure utility. It takes an input, does some work, and gives an output. It doesn't care about the rest of the class."**

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

In this code:

```python
def create_checkout_session(order: Order):
  # Some logic
```

I think you are using something like `TypeScript`. What is the equivalent of TypeScript in `Python` world? I've been a frontend developer for three years and I know that in frontend development, using TypeScript is an industry standard and we had to learn it. Does type safety in Python is something we have to learn like in JS world?

## Answer:

In the code `def create_checkout_session(order: Order):`, you correctly spotted the resemblance to TypeScript!

In the Python world, this is called **Type Hinting** (or Type Annotations). As a frontend developer with three years of TypeScript experience, you'll find the concepts very familiar, but the "philosophy" behind it is slightly different.

---

### 1. Is it exactly like TypeScript?

**Almost.** TypeScript is a superset of JavaScript that **compiles** into JS. Python's type hints are a built-in syntax (starting from Python 3.5) that stay inside your code.

| Feature         | TypeScript (Frontend)                                                   | Type Hinting (Python)                                                          |
| --------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Enforcement** | **Strict:** If types are wrong, the code won't compile to JS (usually). | **Optional:** Python will run the code even if the types are completely wrong. |
| **Runtime**     | Types are stripped away (they don't exist in the browser).              | Types exist but are ignored by the Python "engine" at runtime.                 |
| **Tooling**     | Built-in to `tsc` and IDEs.                                             | Requires an external tool like **Mypy** or **Pyright** to "check" the code.    |

---

### 2. Is it an "Industry Standard" you have to learn?

**Yes.** While Python is technically "dynamically typed" (meaning you don't _have_ to use types), any professional Django or FastAPI project in 2025 uses Type Hinting.

For you, coming from a frontend background, this is great news because:

1. **Autocomplete (IntelliSense):** In VS Code or PyCharm, if you type `order.`, the IDE will know exactly what fields are on the `Order` model.
2. **Safety:** If you accidentally pass a `User` object instead of an `Order` object, your editor will underline it in red before you even run the server.
3. **Documentation:** It tells your teammates exactly what a function needs without them having to read the whole function body.

---

### 3. How do you "enforce" it?

In TypeScript, the "check" happens when you run `npm run build`. In Python, we use a tool called **Mypy**. You can run it in your terminal:

```bash
pip install mypy
mypy your_project_folder/

```

If you passed a string where an `Order` was expected, Mypy will throw an error, much like `tsc` does in your frontend workflow.

---

### 4. Comparison of Syntax

Since you know TypeScript, you can basically "guess" 90% of Python typing:

| Feature               | TypeScript Syntax | Python Syntax      |
| --------------------- | ----------------- | ------------------ | -------- | ---- |
| **Basic Type**        | `name: string`    | `name: str`        |
| **Lists**             | `items: string[]` | `items: list[str]` |
| **Optional/Nullable** | `name?: string`   | `name: str         | None`    |
| **Multiple Types**    | `id: number       | string`            | `id: int | str` |

### Why I used it in your `services.py`

In professional Django development, the `services.py` layer is where the most complex business logic happens. By typing `order: Order`, I am ensuring that we don't accidentally send the wrong data to Stripe, which could cause a payment to fail and lose you money.

## Question:

I have some questions about the `create` method in `stripe.checkout.Session`:

```python
session = stripe.checkout.Session.create(
  payment_method_types=["card"],
  line_items=[
  {
   "price_data": {
      "currency": "usd",
      "product_data": {
         "name": f"Order #{order.id}",
                      },
      "unit_amount": int(order.total_paid * 100),  # Stripe uses cents
                 },
  "quantity": 1,
  }
                ],
```

My first question is that I have seen this in `Stripe` documentation:

```text
line_items.price_data
dictionary
Required conditionally
Data used to generate a new Price object inline. One of price or price_data is required.
```

First tell me what does documentation mean when it says `Required conditionally`?

## Answer:

In technical documentation, **"Required Conditionally"** is a fancy way of saying: _"This field is mandatory, but only if you haven't provided a specific alternative."_

It’s like a form that asks for your "Phone Number" or "Email." Neither is strictly required on its own, but you **must** provide at least one so they can contact you.

### Why Stripe uses this for `price_data`

When you create a checkout session, Stripe needs to know **how much to charge**. There are two ways to tell it:

1. **Option A: The `price` ID:** You have already created a product (like "Gold Plan") in your Stripe Dashboard. You just send the ID (e.g., `price_123...`).
2. **Option B: The `price_data` dictionary:** You want to create a price "on the fly" (ad-hoc) because the price changes for every order (e.g., a custom invoice or a dynamic cart).

**The Condition:**

- If you **do not** provide a `price` ID, then `price_data` becomes **Required**.
- If you **do** provide a `price` ID, then `price_data` must be **Omitted** (otherwise Stripe gets confused).

---

### Understanding your code vs. the Dashboard

In the code you shared, you are using the **"On the fly" (price_data)** approach:

```python
line_items=[{
    "price_data": { # <--- This is REQUIRED because we aren't using a "price" ID
        "currency": "usd",
        "product_data": {
            "name": f"Order #{order.id}",
        },
        "unit_amount": int(order.total_paid * 100),
    },
    "quantity": 1,
}]

```

### When should you use which?

| Use Case              | Use `price` (ID)                                  | Use `price_data` (Inline)                |
| --------------------- | ------------------------------------------------- | ---------------------------------------- |
| **SaaS Subscription** | **Yes.** You have fixed plans ($10, $20, $50).    | No.                                      |
| **E-commerce Store**  | **Yes.** For physical products with fixed prices. | No.                                      |
| **Custom Invoices**   | No.                                               | **Yes.** Every order has a unique price. |
| **Donations**         | No.                                               | **Yes.** User chooses how much to pay.   |

---

### A Note on `product_data`

Notice that inside `price_data`, there is a field called `product_data`. This is **also conditionally required**.

- If you have an existing Product ID in Stripe, you use `price_data: {"product": "prod_123"}`.
- If you are creating a brand new product name right now, you use `price_data: {"product_data": {"name": "..."}}`.

## Question:

So based on your answer and what I saw on the `Stripe` documentation, it seems we can create products and prices on the Stripe dashboard, but for our E-Commerce project and implementation, we don't use this feature. Tell me about this feature.

## Answer:

In a professional E-Commerce project, you have a choice: you can keep your products in your **own database** (Django) or store them in **Stripe's database** (the Product Catalog).

Since you are a frontend developer, think of this like **State Management**:

- **`price_data` (Your current way):** Your Django database is the "Single Source of Truth." Stripe just follows orders.
- **Stripe Product Catalog:** Stripe becomes the "Source of Truth" for your products.

### 1. What is the Stripe Product Catalog?

It is a simplified CMS (Content Management System) built into the Stripe Dashboard. You can manually click "Add Product," upload an image, and set a price.

When you do this, Stripe gives you a **Price ID** (e.g., `price_1PqR3...`). Instead of sending a giant dictionary of details to Stripe, you just send that one ID.

---

### 2. Why _wouldn't_ you use it for an E-Commerce site?

For a standard store, keeping products in your own Django database (as you are doing) is usually the better choice. Here is why:

- **Inventory Control:** Stripe doesn't know how many blue t-shirts are left in your warehouse. Your Django database does.
- **Complex Relationships:** You might have "Buy 1 Get 1 Free" deals or user-specific discounts. It is much easier to calculate the final price in Python and send it to Stripe via `price_data`.
- **SEO & Content:** Your frontend needs a database to show product pages, search results, and categories. If you stored everything in Stripe, you'd have to make an API call to Stripe just to show a "Product Details" page, which is slow.

---

### 3. When is this feature actually useful?

The Dashboard catalog is "Industry Standard" for **SaaS (Software as a Service)** and **Subscriptions**.

- **Scenario:** You sell a "Pro Plan" for $20/month.
- **The Benefit:** You define it once in the Dashboard. If you decide to raise the price to $25, you change it in the Dashboard, and _every_ user's next bill automatically updates. You don't have to touch a single line of code in your backend.
- **No-Code Tools:** It allows marketing teams to create "Payment Links" (a simple URL you can put in an Instagram bio) without needing a developer to build a checkout page.

---

### 4. Comparison: Dashboard IDs vs. Your `price_data`

| Feature            | Your Way (`price_data`)                                   | Dashboard Way (`price_id`)                                 |
| ------------------ | --------------------------------------------------------- | ---------------------------------------------------------- |
| **Flexibility**    | **Infinite.** You can charge $99.99 or $12.34 on the fly. | **Fixed.** You can only charge the prices you pre-defined. |
| **Maintenance**    | Code handles everything.                                  | You must sync Stripe IDs with your DB.                     |
| **Best For**       | E-commerce with 100s of unique carts.                     | Subscriptions and fixed digital services.                  |
| **Dashboard View** | Payments show up as "Order #123".                         | Payments show up as "1x Blue T-Shirt".                     |

### Summary

In your project, you are using Stripe as a **"dumb" payment terminal**. Your Django backend is the "brain" that knows the prices. This is the correct architecture for an e-commerce platform because it gives you total control over your business logic.

## Question:

For `payment_method_types` attribute, what are other options than "card"?

## Answer:

While `"card"` is the most universal option, Stripe supports dozens of payment methods to accommodate different global habits, currencies, and trust levels.

In modern Stripe integrations, however, the way you use this attribute has actually changed.

### 1. Common Alternatives to "card"

Depending on where your customers are located, you might use:

- **Digital Wallets:** `"apple_pay"`, `"google_pay"`, `"link"` (Stripe’s own 1-click checkout).
- **Bank Redirects (Europe):** `"ideal"` (Netherlands), `"bancontact"` (Belgium), `"giropay"` (Germany).
- **Bank Debits (US/UK):** `"us_bank_account"` (ACH), `"bacs_debit"`.
- **Buy Now, Pay Later (BNPL):** `"klarna"`, `"afterpay_clearpay"`.
- **Real-time Payments:** `"pix"` (Brazil), `"paynow"` (Singapore).

---

### 2. The "Modern" Way: Automatic Payment Methods

In older code, developers had to manually list every string in that array:
`payment_method_types=["card", "ideal", "klarna"]`.

**The Problem:** If you wanted to add a new method, you had to change your code, redeploy your server, and test it again.

**The Solution:** Most developers now use **Automatic Payment Methods**. Instead of `payment_method_types`, you use this in your `create` method:

```python
session = stripe.checkout.Session.create(
    # payment_method_types is REMOVED
    automatic_payment_methods={"enabled": True},
    # ... rest of your code
)

```

### Why use Automatic Payment Methods?

1. **No Code Changes:** You can go into your **Stripe Dashboard Settings**, click a toggle to "Enable Klarna," and it will instantly appear on your checkout page without you touching your Django code.
2. **Smart Sorting:** Stripe uses AI to show the most relevant methods first. If a user is on an iPhone in the UK, it will prioritize Apple Pay and BACS. If they are in the US, it shows Credit Cards and ACH.
3. **Currency Matching:** It only shows payment methods that support the currency you are charging in.

---

### 3. When should you still use `payment_method_types`?

You only hardcode the array (like in your current code) if you want **strict control**. For example:

- You are a high-risk business and only want to accept "Non-refundable" methods.
- You want to exclude "Buy Now Pay Later" options because their transaction fees are higher (often 5-6% instead of 2.9%).
- You are testing a very specific local payment flow.

### Summary Table: Popular Methods

| Type              | String ID           | Why use it?                                                 |
| ----------------- | ------------------- | ----------------------------------------------------------- |
| **Credit/Debit**  | `"card"`            | Universal, instant.                                         |
| **Bank Transfer** | `"us_bank_account"` | Much lower fees for large transactions (capped at $5).      |
| **Installments**  | `"klarna"`          | Increases conversion for expensive items ($500+).           |
| **Wallets**       | `"link"`            | 1-click checkout for customers who have used Stripe before. |

## Question:

Consider this code:

```python
success_url=settings.PAYMENT_SUCCESS_URL + "?session_id={CHECKOUT_SESSION_ID}",
```

Why you added this part:
`?session_id={CHECKOUT_SESSION_ID}`

## Answer:

This is a great catch. That specific string—`{CHECKOUT_SESSION_ID}`—is a **template variable** provided by Stripe.

It acts like a "placeholder" that Stripe automatically fills in with the real ID of the transaction _after_ the payment is done, but _before_ the user lands back on your site.

### 1. Why do we need it? (The "Context" Problem)

When Stripe redirects a user back to your site, your Frontend (React/Vue) is essentially "waking up" again. It doesn't know what happened while the user was away on the Stripe page.

By adding this ID to the URL, your Success Page can "read" the ID from the address bar (the query parameters) and use it to do two things:

- **Retrieve Order Details:** Your frontend can send that ID back to your Django API to say, _"Hey, the user just came back with this Session ID. Is the order actually paid? What did they buy so I can show a 'Thank You' message?"_
- **Verification:** It allows you to confirm that the user actually completed a real session, rather than just manually typing `mysite.com/success` into their browser to try and get free stuff.

---

### 2. How the "Magic" works

Notice that in your Python code, you are writing the literal string `{CHECKOUT_SESSION_ID}` inside the quotes. You are **not** using a Python f-string or variable there.

1. **You send:** `https://mysite.com/success?session_id={CHECKOUT_SESSION_ID}`
2. **User pays:** They finish their credit card info on Stripe.
3. **Stripe Redirects:** Stripe sees that curly-bracket string and replaces it with the actual ID.
4. **User lands on:** `https://mysite.com/success?session_id=cs_test_a1b2c3d4...`

---

### 3. Synchronous vs. Asynchronous Confirmation

This creates two ways for your app to find out a payment was successful:

| Method          | Where it happens             | Name             | Reliability                    |
| --------------- | ---------------------------- | ---------------- | ------------------------------ |
| **Webhook**     | Backend (Stripe -> Django)   | **Asynchronous** | **100%** (It always happens)   |
| **Success URL** | Frontend (Browser -> Django) | **Synchronous**  | **90%** (User might close tab) |

**Why do both?**
We use the **Webhook** to update the database (the source of truth). We use the **Success URL** with the `session_id` to give the user immediate feedback on the UI (the "Thank You" screen) without making them wait for the webhook to finish processing.

---

### 4. How you'll use it in the Frontend

As a frontend developer, you will write a small piece of code on your Success page that looks like this:

```javascript
// On your Success Page component
const urlParams = new URLSearchParams(window.location.search);
const sessionId = urlParams.get("session_id");

if (sessionId) {
  // Call your Django API to verify and show order details
  fetch(`/api/payments/verify-status/?session_id=${sessionId}`)
    .then((res) => res.json())
    .then((data) => {
      if (data.status === "paid") {
        showConfetti(); // Example UI feedback
      }
    });
}
```

## Question:

In `Stripe` documentation, it says `client_reference_id` is nullable. Does it mean it's not required? Can I remove it?

## Answer:

In short: **No, you should not remove it.**

While the documentation says it is "nullable" (meaning Stripe's database can store it as `null`), it is a **crucial** field for your backend logic.

### 1. What does "Nullable" mean here?

In Stripe's API, "nullable" simply means that Stripe won't throw an error if you leave it out. It is **optional** from Stripe's perspective, but it is **essential** for your Django app.

### 2. Why you absolutely need it

Think of `client_reference_id` as the **"Order ID"** in your system. When you redirect a user to Stripe, Stripe doesn't know anything about your database.

Without this ID:

1. **The Webhook Problem:** When the payment is successful, Stripe sends a webhook to your server. Without the `client_reference_id`, your Django app will receive a "Success" message but won't know **which order** was paid for.
2. **Reconciliation:** You use this ID to "reconcile" (match) Stripe's transaction with the specific `Order` record in your Django database.

### 3. How it looks in your workflow

If you remove it, your `fulfill_order` logic will break because it won't have a reference to search for in your database.

```python
# In your create_checkout_session service:
session = stripe.checkout.Session.create(
    client_reference_id=str(order.id),  # <--- THIS IS YOUR KEY
    # ... other settings
)

# Later, in your Webhook view:
if event['type'] == 'checkout.session.completed':
    session = event['data']['object']
    order_id = session.get('client_reference_id') # <--- YOU GRAB IT BACK HERE
    order = Order.objects.get(id=order_id)
    order.status = 'paid'
    order.save()

```

### 4. Summary: Required vs. Nullable

| Term         | Stripe's View                 | Your Project's View                        |
| ------------ | ----------------------------- | ------------------------------------------ |
| **Required** | No (API won't crash)          | **Yes** (Database won't update without it) |
| **Nullable** | Yes (Can be `null` in Stripe) | **No** (Should always be your Order ID)    |

**Recommendation:** Keep it exactly as we wrote it. It is the "glue" that connects Stripe back to your Django database.

## Question:

About this code:

```python
@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META.get("HTTP_STRIPE_SIGNATURE")
    event = None

    try:
        # 1. Verify that the request actually came from Stripe
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        return HttpResponse(status=400)  # Invalid payload
    except stripe.error.SignatureVerificationError:
        return HttpResponse(status=400)  # Invalid signature

    # 2. Handle the specific event
    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]

        # Call our service to fulfill the order
        # This updates the Transaction and the Order status
        success = PaymentService.fulfill_order(session)

        if not success:
            return HttpResponse(status=500)

    # 3. Always return a 200 OK to Stripe so they stop retrying
    return HttpResponse(status=200)
```

Is it a view? I think it's a function-based view. And also tell me what is this `@csrf_exempt` decorator?

## Answer:

You are spot on. This is a **Function-Based View (FBV)**, and it is the most common way to handle webhooks in Django.

### 1. Yes, it is a Function-Based View

In Django, a view is simply a function (or class) that takes a `request` and returns a `response`.

- **The Input:** It receives an `HttpRequest` object containing the data Stripe sent (the payload and the signature).
- **The Logic:** it verifies the data and updates your database.
- **The Output:** It returns an `HttpResponse` with a status code (like `200` or `400`) to let Stripe's server know if the message was received.

---

### 2. What is `@csrf_exempt`?

Here is the meaning of the word `exempt`:

`/ɪɡˈzempt/ ●○○ adjective`  
`not affected by something, or not having to do it or pay it`

Examples:

```
exempt from

The interest is exempt from income tax.
Children are exempt from the charges.
```

To understand this decorator, we have to look at how Django protects your site.

**The Background: CSRF Protection**
Django has a built-in security feature called **Cross-Site Request Forgery (CSRF)** protection. For every `POST` request, Django expects a "CSRF Token"—a secret key that proves the request is coming from _your_ own frontend. If a `POST` request arrives without this token, Django blocks it and returns a `403 Forbidden` error.

**The Problem with Webhooks**
When Stripe sends a webhook, it is an external server making a `POST` request to your backend. Stripe does not (and cannot) have your site's secret CSRF token.

- If you **don't** use `@csrf_exempt`, Django will reject every single notification from Stripe, thinking it is a hacker attack.

**The Solution: `@csrf_exempt**`
This decorator tells Django: _"For this specific view only, turn off the CSRF token check. I trust that I have a different way to verify the requester."_

---

### 3. Is it safe to turn off CSRF?

**Yes, in this specific case.** Even though we are turning off Django's standard CSRF protection, we are replacing it with something **even stronger** inside the function:

```python
# This part replaces CSRF protection
event = stripe.Webhook.construct_event(
    payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
)

```

The `construct_event` method uses a **cryptographic signature**. It checks a special header (`sig_header`) against your `STRIPE_WEBHOOK_SECRET`.

- If a hacker tries to send a fake "Payment Success" request to this URL, they won't have your secret key.
- The verification will fail, and your code will return a `400` error before any database changes happen.

### Summary

- **Function-Based View:** Chosen because webhooks are usually simple, one-off logic flows that don't need the complexity of a class.
- **`@csrf_exempt`:** Necessary because Stripe is an "outside" server that doesn't have your internal Django security tokens.

## Question:

Tell me briefly, is it better to use a `FBV` or a `CBV` for this `webhook`? Also tell me why you keep saying "Stripe sends a webhook"? Is a webhook something we can send? Tell me again what a webhook is.

## Answer:

To answer your first question: for a Stripe webhook, a **Function-Based View (FBV)** is generally better and much more common in the industry.

### 1. FBV vs. CBV for Webhooks

While you _can_ use a Class-Based View (CBV), most developers stick to a **Function-Based View** for webhooks because:

- **Simplicity:** Webhooks usually only handle one type of request: a `POST`. A function is the most direct way to handle a single action.
- **Decorator Ease:** Applying `@csrf_exempt` is slightly more annoying on a class (you have to use `method_decorator` on the `dispatch` method). On a function, you just drop the decorator on top and you're done.
- **Readability:** Webhook logic is often a long list of `if/elif` statements checking event types. Keeping this in a flat function makes it very easy to read.

---

### 2. Why do I say "Stripe sends a webhook"?

I use that phrase because, in this relationship, **Stripe is the sender** and **you are the receiver**.

A "Webhook" is just an automated message sent from an app when something happens.

- **Can you send a webhook?** Yes! If you built an app that other developers used (like a shipping platform), you would "send a webhook" to _their_ servers whenever a package was delivered.
- **In this project:** You are the one waiting for the news. You are "consuming" the webhook that Stripe sends.

---

### 3. What is a Webhook? (The "Mailbox" Analogy)

Think of a **Standard API** (like your Product List) as a **Vending Machine**:
You go to it, you press a button (make a request), and it gives you a soda (the data). If you don't go to the machine, nothing happens.

Think of a **Webhook** as a **Mailbox**:
You don't go anywhere. You just sit at home. When the mailman (Stripe) has a letter for you (the payment confirmation), he drives to your house and puts it in your mailbox.

#### Key Characteristics:

1. **Event-Driven:** It only triggers when an "event" happens (Payment Success, Refund, Subscription Cancelled).
2. **Push, Not Pull:** Stripe "pushes" the data to you. You don't "pull" it from Stripe.
3. **One-Way:** Stripe sends the data, your server says "Got it (200 OK)," and the conversation ends.

---

### 4. The Webhook Loop

1. **Setup:** You tell Stripe, "Hey, if anyone pays me, send a message to `https://my-site.com/webhook/`."
2. **Trigger:** A customer buys a t-shirt.
3. **Action:** Stripe's server automatically builds a JSON package and "POSTs" it to your URL.
4. **Reaction:** Your Django function wakes up, sees the data, and updates your database.

## Question:

So again tell me why I need this `@csrf_exempt` decorator knowing that I use `DRF` and `Simple JWT` for authentication.

## Answer:

The reason you still need `@csrf_exempt` is because of a "clash" between **Django’s global security** and **Stripe’s external nature.**

Even though you are using Simple JWT for your users, the Webhook is a completely different animal. Here is exactly why the decorator is mandatory for that one specific function:

### 1. The Global Middleware "Wall"

Django has a list called `MIDDLEWARE` in your `settings.py`. One of them is `django.middleware.csrf.CsrfViewMiddleware`.

This middleware is like a security guard standing at the front gate of your entire application. He checks **every single** `POST` request that enters your site, regardless of whether it's a DRF view or a plain Django view.

If a request doesn't have a CSRF token, the guard throws it out **before it even reaches DRF's authentication logic.**

---

### 2. Stripe cannot "Log In"

When you use your app, your frontend sends a JWT. DRF sees that JWT and says, "I know this person; skip the CSRF check."

However, when Stripe sends a webhook:

- **It has no JWT:** Stripe doesn't have an account on your site.
- **It has no CSRF Token:** Stripe doesn't know your site's secret tokens.
- **It is an "Anonymous" POST:** To your Django middleware, the Stripe request looks like a suspicious, anonymous `POST` request from the outside world.

Without `@csrf_exempt`, the Middleware guard will block Stripe's request immediately, and your `stripe_webhook` function will never even execute.

---

### 3. Safety: Signature vs. CSRF

You might feel like you are leaving a "hole" in your security by using `@csrf_exempt`. But you aren't. You are simply swapping one type of lock for another.

- **CSRF** is designed to prove a request came from **your frontend**.
- **Stripe Signatures** are designed to prove a request came from **Stripe's servers**.

By using `@csrf_exempt`, you tell the Middleware guard: _"Let this one guest in without a token. I have my own specialized guard (Stripe's Signature Verifier) inside the room who will check his ID."_

---

### Summary Table

| Request Type               | Auth Method    | Why it bypasses CSRF                                                                                              |
| -------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------- |
| **User clicking "Pay"**    | **Simple JWT** | DRF's `JWTAuthentication` tells Django it's safe because it's a header-based token.                               |
| **Stripe sending Webhook** | **Signature**  | **`@csrf_exempt`** tells Django's Middleware to step aside so your code can verify the Stripe signature manually. |

### The "Backend Dev" Pro-Tip

If you ever forget `@csrf_exempt` on a webhook, you will see `403 Forbidden` errors in your Stripe Dashboard's "Webhook Events" log, but you won't see any errors in your Django console. This is the #1 reason webhook integrations fail during the first attempt!

## Question:

In our `stripe_webhook` view, we have this line:

```python
sig_header = request.META.get("HTTP_STRIPE_SIGNATURE")
```

What is this `META` dictionary? Does it include the HTTP headers?

## Answer:

Yes, you are exactly right. In Django, `request.META` is a standard Python dictionary that contains all available **HTTP headers** for the current request, along with several pieces of server-specific information.

### 1. What is in the META dictionary?

When a request hits your Django server, it includes metadata (data about the data). This dictionary contains:

- **HTTP Headers:** Any header sent by the client (or Stripe). In Django, these are converted to uppercase, dots/hyphens become underscores, and they are prefixed with `HTTP_`.
- Example: A header sent as `Stripe-Signature` becomes `HTTP_STRIPE_SIGNATURE`.

- **Server Variables:** Information like `REMOTE_ADDR` (the IP address of the sender), `SERVER_NAME`, and `REQUEST_METHOD` (`POST`, `GET`, etc.).

---

### 2. Why does Stripe use a custom header?

Stripe sends the `Stripe-Signature` header to provide a way for you to verify that the message hasn't been tampered with. Because you are `@csrf_exempt`, this header is your **new security guard**.

---

### 3. How Django transforms headers

As a backend developer, it's important to remember the "naming convention" Django uses for the `META` dictionary. Here is a quick reference table:

| Original Header (Sent by Stripe) | Django META Key                                 |
| -------------------------------- | ----------------------------------------------- |
| `Content-Type`                   | `CONTENT_TYPE` (Special case, no HTTP prefix)   |
| `Content-Length`                 | `CONTENT_LENGTH` (Special case, no HTTP prefix) |
| `Stripe-Signature`               | `HTTP_STRIPE_SIGNATURE`                         |
| `User-Agent`                     | `HTTP_USER_AGENT`                               |
| `Authorization`                  | `HTTP_AUTHORIZATION`                            |

---

### 4. Why use `.get()`?

In the code `request.META.get("HTTP_STRIPE_SIGNATURE")`, we use `.get()` instead of square brackets `[]` for safety.

- If a hacker (or a random bot) hits your webhook URL without the signature header, `request.META["HTTP_STRIPE_SIGNATURE"]` would crash your server with a `KeyError`.
- Using `.get()` returns `None` instead of crashing, allowing your `try/except` block to handle the error gracefully and return a `400 Bad Request`.

---

### Summary for the Backend Dev

`request.META` is your "eyes" into the raw HTTP details. It tells you **who** is calling (IP address) and **how** they are proving their identity (Headers). For webhooks, the `HTTP_STRIPE_SIGNATURE` is the most important piece of data in that entire dictionary.

## Question:

What does this code do:

```python
event = stripe.Webhook.construct_event(payload,     sig_header, settings.STRIPE_WEBHOOK_SECRET)
```

## Answer:

This line of code is the **security checkpoint** for your webhook. It is the most important part of the entire integration because it prevents hackers from "faking" successful payments.

Essentially, it takes three ingredients, mixes them together using cryptography, and tells you: _"Yes, this message definitely came from Stripe and hasn't been changed."_

Here is the breakdown of what is happening inside that function:

### 1. The Three Ingredients

To verify the message, the Stripe SDK needs three pieces of information:

- **`payload`**: The raw body of the request (the JSON message).
- **`sig_header`**: The signature found in `request.META`. This is what Stripe "signed" the message with.
- **`settings.STRIPE_WEBHOOK_SECRET`**: Your "Secret Key" (only known by you and Stripe). This is the "key" used to unlock and verify the signature.

---

### 2. What it actually "Does" (The Logic)

Under the hood, this function performs **Signature Verification**. Here is the step-by-step process:

1. **Re-calculates the Hash:** The function takes the `payload` and your `SECRET` and runs a math formula (HMAC-SHA256) to create a unique code (a hash).
2. **Compares the Codes:** It looks at the `sig_header` sent by Stripe and compares it to the hash it just calculated.
3. **Checks the Timestamp:** It also checks a timestamp inside the header to ensure the request isn't an old one being "replayed" by a hacker (to prevent "Replay Attacks").

---

### 3. The Possible Outcomes

This function acts as a gatekeeper. Depending on the result, one of two things happens:

- **Success:** If the math matches, the function returns an **Event Object**. This is a clean, verified Python dictionary containing the payment details. You can now safely update your database.
- **Failure:** If the math doesn't match (e.g., if a hacker changed the price in the payload or used the wrong key), the function **raises an exception** (`SignatureVerificationError`). Your `try/except` block will catch this and stop the code from running further.

---

### 4. Why can't we just use `json.loads(request.body)`?

As a backend developer, you might be tempted to just parse the JSON directly. **Never do this for webhooks.** If you just use `json.loads()`:

- A hacker could find your webhook URL.
- They could send a fake POST request saying `{"status": "paid", "order_id": 99}`.
- Your server would believe them and ship the product for free.

By using `construct_event`, you ensure that **only Stripe** can trigger your fulfillment logic.

---

### 5. Summary Table

| Input          | Role                                                  |
| -------------- | ----------------------------------------------------- |
| **Payload**    | The "Letter" (What happened?)                         |
| **Sig_header** | The "Wax Seal" on the envelope (Who sent it?)         |
| **Secret**     | The "Decoder Ring" (How do I prove the seal is real?) |

## Question:

I saw this sample code in `Stripe` documentation:

```python
import json
from django.http import HttpResponse

# Using Django
# Replace this endpoint secret with your unique endpoint secret key
# If you're testing with the CLI, run 'stripe listen' to find the secret key
# If you defined your endpoint using the API or the Dashboard, check your webhook settings for your endpoint secret: https://dashboard.stripe.com/webhooks
endpoint_secret = 'whsec_...'

@csrf_exempt
def my_webhook_view(request):
  payload = request.body
  event = None

  try:
    event = stripe.Event.construct_from(
      json.loads(payload), stripe.api_key
    )
  except ValueError as e:
    # Invalid payload
    return HttpResponse(status=400)

  if endpoint_secret:
        # Only verify the event if you've defined an endpoint secret
        # Otherwise, use the basic event deserialized with JSON
        sig_header = request.headers.get('stripe-signature')
        try:
            event = stripe.Webhook.construct_event(
                payload, sig_header, endpoint_secret
            )
        except stripe.error.SignatureVerificationError as e:
            print('⚠️  Webhook signature verification failed.' + str(e))
            return jsonify(success=False)

  # Handle the event
  if event.type == 'payment_intent.succeeded':
    payment_intent = event.data.object # contains a stripe.PaymentIntent
    # Then define and call a method to handle the successful payment intent.
    # handle_payment_intent_succeeded(payment_intent)
  elif event.type == 'payment_method.attached':
    payment_method = event.data.object # contains a stripe.PaymentMethod
    # Then define and call a method to handle the successful attachment of a PaymentMethod.
    # handle_payment_method_attached(payment_method)
  # ... handle other event types
  else:
    print('Unhandled event type {}'.format(event.type))

  return HttpResponse(status=200)
```

Why it uses `request.headers` but you told me to use `request.META`:

```python
sig_header = request.headers.get('stripe-signature')
```

## Answer:

This is a perfect example of the "Modern Django" vs. "Classic Django" distinction I mentioned earlier.

The short answer is: **They both work, but `request.headers` is the modern, more readable way.**

Here is exactly why you see both versions in documentation and tutorials:

### 1. The History (Django 2.2+)

For a very long time (since 2005), `request.META` was the **only** way to get headers in Django. Because it was the only way for 15 years, many older Stripe tutorials and code snippets still use it.

In **Django 2.2** (released in 2019), the Django team added the `request.headers` object specifically because developers hated dealing with the weird `HTTP_` prefix and capitalization of `request.META`.

### 2. A Comparison of Syntax

If Stripe sends a header named `Stripe-Signature`, here is how you access it in the two versions:

| Method                 | Syntax                                      | Pros/Cons                                                                                                                         |
| ---------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Classic (`META`)**   | `request.META.get('HTTP_STRIPE_SIGNATURE')` | **Pro:** Works on ancient versions of Django. **Con:** Hard to read, non-standard naming.                                         |
| **Modern (`headers`)** | `request.headers.get('Stripe-Signature')`   | **Pro:** Very readable, matches exactly what you see in the Stripe Dashboard. **Con:** None (unless you're on a very old server). |

---

### 3. Which one should YOU use?

Since you are using a modern version of Django and DRF:

- **Use `request.headers**` for your own custom logic because it’s cleaner and less confusing.
- **Keep an eye on the Stripe SDK:** In some older versions of the `stripe-python` library, their internal verification methods specifically looked for the `META` dictionary format. However, the modern Stripe SDK (like the one in the code you just shared) works perfectly fine with a simple string from `request.headers`.

---

### 4. A small catch in the sample code

Did you notice this in the sample code?

```python
sig_header = request.headers.get('stripe-signature')

```

Notice it's all **lowercase**.
While HTTP headers are technically case-insensitive, Django's `request.headers` is a "Case-Insensitive Dict." This means `stripe-signature`, `Stripe-Signature`, and `STRIPE-SIGNATURE` will all return the same value. This makes your life much easier as a backend dev!

---

### 5. Why the sample code looks more complex

The sample code you found is doing something extra. It first tries to "construct" the event from JSON, and _then_ verifies it with the signature.

```python
# The sample logic:
1. Parse the JSON body (event = stripe.Event.construct_from(...))
2. IF we have a secret, RE-VERIFY it (event = stripe.Webhook.construct_event(...))

```

**My advice:** In your real production code, don't bother with the first step (`construct_from`). Just go straight to `stripe.Webhook.construct_event`. If the signature is valid, you know the event is valid. There is no need to do the work twice!

### Summary

- **`request.headers`** is the "New Way" (User-friendly).
- **`request.META`** is the "Old Way" (System-friendly).
- As a modern Django developer, **embrace `request.headers**`. It will make your code look much cleaner to other developers.

## Question:

You told me this:

Django's `request.headers` is a `Case-Insensitive Dict.`

Are dictionaries in Python `case-insensitive`? Or we can create `case-insensitive` dictionaries?

## Answer:

Standard Python dictionaries are **not** case-insensitive. If you try to access a key with different casing in a normal dict, Python will treat them as two completely different entries (or throw a `KeyError`).

However, Django created a special **class** that behaves like a dictionary but ignores casing. This is what you are using when you call `request.headers`.

### 1. Standard Python Dictionaries (Case-Sensitive)

In regular Python, keys must match exactly.

```python
# Standard Dict
user = {"Name": "Alice"}

print(user["Name"])  # Output: Alice
print(user["name"])  # Raises KeyError!

```

---

### 2. Can we create Case-Insensitive Dictionaries?

Yes! In Python, you can create your own custom dictionary behavior by **inheriting** from `dict` or `collections.UserDict` and overriding how keys are handled.

To make a dictionary case-insensitive, you would tell it to "convert every key to lowercase" before storing it or looking it up.

**A simple example of how Django does this under the hood:**

```python
class MyCaseInsensitiveDict(dict):
    def __getitem__(self, key):
        # Always look up the lowercase version of the string
        return super().__getitem__(key.lower())

    def __setitem__(self, key, value):
        # Always store the lowercase version of the string
        super().__setitem__(key.lower(), value)

# Usage
headers = MyCaseInsensitiveDict()
headers["Content-Type"] = "application/json"

print(headers["content-type"]) # Output: application/json
print(headers["CONTENT-TYPE"]) # Output: application/json

```

---

### 3. Why Django uses `HttpHeaders`

HTTP headers are defined by the official web standards as **case-insensitive**. Whether a browser sends `Authorization`, `authorization`, or `AUTHORIZATION`, the server is supposed to treat them as the same thing.

Django’s `request.headers` is an instance of `django.utils.datastructures.CaseInsensitiveMapping`.

This is a huge relief for you as a backend developer because:

1. You don't have to guess if the frontend developer used `lowercase` or `Title Case`.
2. You don't have to worry about different browsers (Chrome vs. Safari) sending headers with different casing.
3. Your code is much more robust and less likely to crash over a simple string mismatch.

---

### 4. Important Distinction: `request.META`

Remember our talk about the "Old Way"? Unlike `request.headers`, **`request.META` is a standard Python dictionary.**

- `request.META["HTTP_STRIPE_SIGNATURE"]` **will work.**
- `request.META["http_stripe_signature"]` **will fail.**

This is another reason why I recommended moving toward `request.headers`. It handles the annoying "exact match" requirement of standard Python dicts for you.

### Summary

- **Standard Python:** Dicts are case-sensitive.
- **Django's `request.headers`:** It is a special "Case-Insensitive Mapping" designed to follow HTTP standards.
- **The Benefit:** It makes your backend code more "forgiving" and easier to maintain.

## Question:

Describe the frontend user workflow in an e-commerce platform after he adds some products to his cart and wants to finalize his order.

## Answer:

1. **Go to cart**

- User lands on the Cart page and can freely edit the basket: change quantities, remove items, apply/clear a cart-level coupon (optional), continue shopping.
- Nothing is “locked” yet; this is still a flexible, pre-purchase state.
- Main actions: **Continue shopping** or **Checkout**.

2. **Checkout (on your site)**

- User enters the checkout flow (often 1–3 screens, sometimes one page).
- You collect/confirm the purchase details needed to price the order correctly:

  - shipping address
  - shipping method
  - billing details (sometimes)
  - taxes/VAT calculations
  - discounts/coupons
  - contact email/phone

- You show the **final totals** (items + shipping + tax − discounts).
- Main action: **Proceed to payment**.

3. **Proceed to payment**

- This is the moment your frontend calls your backend to:

  - validate the checkout data again (stock, prices, totals)
  - create/update the **Order** (usually from “draft” → “payment_pending”)
  - create **StockReservation holds** (so availability updates immediately)
  - create the **Stripe Checkout Session** (with expiry)

- Your backend returns the Stripe `checkout_url`, and the frontend redirects the user to Stripe.

4. **Pay (on Stripe Checkout)**

- User enters card/payment details and clicks **Pay**.
- Stripe completes (or fails) the payment.
- Stripe redirects the user back to your **success_url** or **cancel_url**.
- Separately (and most importantly), Stripe sends your backend a **webhook**:

  - on success: you mark the order paid, consume reservations, decrement on-hand stock, fulfill
  - on expiration/cancel: you release reservations so stock becomes available again

## Question:

In which phase I should create an order instance?

## Answer:

Create the **Order** at **phase 2: Checkout (on your site)**.

- Cart (phase 1) stays mutable and “informal”.
- When the user clicks **Checkout**, you create an **Order in draft/pending state** (snapshot of cart items).
- When the user clicks **Proceed to payment** (phase 3), you **finalize** the order (totals/tax/shipping), **reserve stock**, and create the Stripe session.

## Question:

Tell me what is `TTL`?

## Answer:

**TTL = Time To Live**

It means:

- how long something is considered **valid**
- after that time, it **expires automatically**

In your case:

- TTL = how long a **StockReservation / Stripe session** is allowed to exist
- after TTL → reservation is released, stock becomes available again

Example:

- TTL = 15 minutes → if user doesn’t pay in 15 min, inventory is freed

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
