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

## Q&A Section

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

What is a webhook and why we need it here?

## Answer:

A **Webhook** is essentially a "reverse API." In a normal API, your application asks a server for information (Polling). In a webhook, the server **pushes** information to your application the moment something happens.

Think of it this way:

- **Polling:** You calling a pizza shop every 5 minutes to ask, "Is my pizza ready?"
- **Webhook:** You giving the pizza shop your phone number and they call you the second the pizza comes out of the oven.

---

### Why do we need it in a Payment App?

In a perfect world, a user pays on the Stripe page and is redirected back to your "Success" page, where you mark the order as paid. But the world isn't perfect.

We need webhooks to handle the **"What Ifs"**:

1. **The "Close Tab" Problem:** A user pays successfully on Stripe, but their internet drops or they accidentally close their browser before the redirect happens. Your website would never know they paid.
2. **The "Slow Bank" Problem:** Some payment methods (like bank transfers or "Buy Now Pay Later") don't succeed instantly. They might take 2 hours or 2 days to clear. Stripe uses a webhook to tell you when the money finally arrives.
3. **Security (The Source of Truth):** You should never trust the frontend to tell you a payment was successful (a hacker could fake a redirect). You only trust a direct, cryptographically signed message from Stripe to your backend.

---

### How it works in our code

1. **Event Occurs:** User finishes payment on the Stripe-hosted page.
2. **Stripe Sends Data:** Stripe's server makes an HTTP `POST` request to your `/api/payments/webhook/` endpoint.
3. **Your Server Acts:** Your Django view receives the data (the "Payload"), verifies it's really from Stripe, and updates your database.

---

### Comparison: Polling vs. Webhook

| Feature         | Polling (Asking)                                           | Webhook (Telling)                                                |
| --------------- | ---------------------------------------------------------- | ---------------------------------------------------------------- |
| **Efficiency**  | **Low:** Wastes resources asking when nothing has changed. | **High:** Only talks to your server when there is news.          |
| **Speed**       | **Delayed:** Depends on how often you ask.                 | **Real-time:** Instant notification.                             |
| **Reliability** | **Risky:** You might stop asking too soon.                 | **Robust:** Stripe will retry for 3 days if your server is down. |

### What happens if your server is down?

This is the best part: If Stripe tries to send a webhook and your server is offline (e.g., you're deploying a new version), Stripe doesn't just give up. It uses **Exponential Backoff**—it will try again in 5 minutes, then 30 minutes, then 2 hours, for up to 3 days until your server says "Got it!"

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
