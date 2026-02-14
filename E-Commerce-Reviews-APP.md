# E-Commerce project reviews app plan

Developing a **reviews app** for a production-grade e-commerce backend requires more than just basic CRUD. You need to handle authentication, ensure data integrity (e.g., preventing a user from reviewing a product they haven't bought), and provide useful data for your frontend (Tailwind/Shadcn).

Since you are using **Django REST Framework (DRF)** and **Simple JWT**, here are the essential and advanced endpoints you should implement.

---

## 1. Public Endpoints (Unauthenticated)

These allow any visitor to see social proof and ratings.

- **`GET /api/reviews/product/{product_id}/`**
- **Purpose:** List all reviews for a specific product.
- **Features:** Implement **Pagination** and **Filtering** (e.g., filter by rating 1–5 stars).

- **`GET /api/reviews/product/{product_id}/stats/`**
- **Purpose:** Get an overview of ratings (Average score, total count, and a breakdown like "5 stars: 80%"). This is vital for building those Shadcn progress bars for rating distributions.

---

## 2. Customer Endpoints (Authenticated)

These require a valid **Simple JWT** and should utilize your custom `User` model.

- **`POST /api/reviews/product/{product_id}/`**
- **Purpose:** Submit a new review.
- **Production Logic:**
- **Permission:** Only allowed if the user has a "Delivered" status in the `orders` app for this specific product (**Verified Purchase**).
- **Validation:** Ensure the user can only leave **one review** per product.

- **`PATCH/PUT /api/reviews/{review_id}/`**
- **Purpose:** Edit an existing review.
- **Logic:** Use a custom permission class (`IsOwnerOrReadOnly`) to ensure users can only edit their own reviews.

- **`DELETE /api/reviews/{review_id}/`**
- **Purpose:** Remove a review.

- **`GET /api/reviews/me/`**
- **Purpose:** Let a user see all reviews they have ever written (useful for a "My Account" dashboard).

---

## 3. Engagement & Interaction (Professional Grade)

To compete with platforms like Amazon, add these "social" features:

- **`POST /api/reviews/{review_id}/vote/`**
- **Purpose:** Mark a review as **"Helpful"** or **"Not Helpful"**.
- **Logic:** Track these in a separate model to prevent double-voting.

- **`POST /api/reviews/{review_id}/report/`**
- **Purpose:** Flag a review for moderation (spam, offensive language).

---

## 4. Admin & Seller Endpoints

- **`GET /api/reviews/pending/`** (Staff Only)
- **Purpose:** A queue for moderators to approve/reject reviews before they go live.

- **`POST /api/reviews/{review_id}/reply/`** (Staff/Seller Only)
- **Purpose:** Allow the store owner to respond to a customer's review (e.g., "We're sorry the fit was small...").

---

### Implementation Tips for Your Stack

1. **Tailwind/Shadcn UI:** When returning the `GET` list, include a `user` object with their `username` and `avatar_url`. This allows you to use the **Shadcn Avatar** and **Card** components effectively.
2. **Star Ratings:** Use an `IntegerChoices` field in your model for the rating (1–5) to make validation easy.
3. **Media:** If you want users to upload photos of the product, ensure your `Review` model has an `ImageField` (or a related `ReviewImage` model) and that you've configured `Media` settings in Django.
4. **Cart Integration:** Use your `cartStore` logic on the frontend to check if a "Review" button should even appear for a user based on their order history.

Would you like me to write the **Django Model** or the **Custom Permission** class to ensure only buyers can leave a review?
