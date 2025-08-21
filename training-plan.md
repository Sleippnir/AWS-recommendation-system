### **Recommendation System: Training Plan & Model Architecture**

This document outlines a draft plan for developing, training, and deploying a two-tower deep learning model for personalized product recommendations, applicable to restaurants, convenience stores, and supermarkets.

### **Phase 1: Data Preparation & Feature Engineering**

The quality of the model is entirely dependent on the quality of the data. This is the most critical phase.

**Objective:** To collect, clean, and preprocess all user, business, product, and interaction data into a feature set ready for model training.

**Key Steps:**

1. **Data Ingestion & Consolidation:**

   * Set up data pipelines to pull raw data from application databases (user profiles, business details, order history, reviews, clicks, saves).

   * Standardize and de-duplicate products and businesses from different sources.

   * Anonymize User Personally Identifiable Information (PII).

2. **Feature Extraction:**

   * **Text Preprocessing:** For user search queries and product reviews, implement text cleaning (lowercase, remove punctuation) and tokenization.

   * **Image Preprocessing:** For all product images, resize them to a standard dimension (e.g., 224x224 pixels) and normalize pixel values.

   * **Categorical Feature Encoding:** Create a vocabulary (a unique integer mapping) for all categorical features (e.g., `category_type`, `price_range`, `brand`).

3. **Offline Feature Generation:**

   * **Generate Image Embeddings:** Using a pre-trained `EfficientNetB0`, process every product image to generate a 1280-dimension vector. Store these in a fast-access database (e.g., a key-value store) mapped by `product_id`.

   * **Generate User Visual Preference Embeddings:** For each active user, retrieve the embeddings of the last 20 product images they positively interacted with and compute the average. Store this as the user's visual preference vector.

   * **Create Training Examples with Bias-Aware Negative Sampling:** Generate the final training dataset. The default approach is a hybrid negative sampling strategy that actively counters popularity bias:

     * **Primary Method (In-Batch Negatives with Correction):** For each positive pair, all other items in the training batch will be used as negatives. To counter popularity bias, apply a correction factor (e.g., log-uniform sampling) that reduces the probability of popular items being selected as negatives. This forces the model to learn from a more diverse set of items, including the long-tail.

     * **Secondary Method (Hard Negative Mining):** Augment the corrected in-batch negatives with a smaller, curated set of hard negatives. These are items that were shown to the user but not clicked or items that are semantically very similar to the positive item.

### **Phase 2: Model Development & Training**

**Objective:** To build and train the two-tower neural network until it can effectively distinguish between good and bad recommendations.

**Key Steps:**

1. **Model Implementation (TensorFlow/PyTorch):**

   * Implement the User Tower and Product/Business Tower architectures as defined below.

   * Use a pre-trained `all-MiniLM-L6-v2` model for text feature extraction.

   * **Loss Function Selection Framework:** The recommended approach is to start with the most efficient method and treat others as subsequent optimizations.

     * **Recommended Baseline: Softmax Cross-Entropy.** This will be the primary loss function used with the hybrid negative sampling strategy. It treats the problem as a multi-class classification task and is the industry standard for its training stability, efficiency, and strong performance.

     * **Potential Future Optimization: Triplet Loss.** This can be explored in later iterations. While it can offer higher precision by directly optimizing the embedding space, it introduces complexity in tuning (e.g., the margin hyperparameter) and is most effective when a very clean and reliable source of hard negatives is established.

2. **Training & Hyperparameter Tuning:**

   * **Initial Training:** Train the model on the prepared dataset. Monitor training/validation loss and accuracy metrics.

   * **Hyperparameter Tuning:** Use a tool like KerasTuner or Optuna to systematically find the best hyperparameters (learning rate, embedding dimension size, dropout rate).

   * **Implement Feature Dropout:** During training, randomly zero-out all behavioral and interaction features for \~20% of the user profiles in each batch. This forces the model to learn how to make recommendations from static profile data alone, effectively solving the cold-start problem.

3. **Model Checkpointing:** Save model weights periodically during training to prevent loss of progress.

### **Phase 3: Evaluation, Deployment & Iteration**

**Objective:** To validate the model's performance, deploy it for serving recommendations, and set up a system for continuous improvement.

**Key Steps:**

1. **Offline Evaluation:**

   * On a held-out test set, evaluate the model using standard retrieval metrics:

     * **Top-K Accuracy:** What percentage of the time is the correct product in the top K recommendations?

     * **Mean Reciprocal Rank (MRR):** Measures the rank of the correct item.

2. **Candidate Indexing:**

   * Use the trained Product/Business Tower to compute the final embeddings for **all** products in the database.

   * Load these embeddings into an Approximate Nearest Neighbor (ANN) search index using **ScaNN**.

3. **Deployment (Retrieval & Re-ranking Pipeline):**

   * Deploy the trained User Tower as a service that can compute a user's embedding in real-time.

   * Create a multi-stage recommendation endpoint that executes the full pipeline:

     1. **Stage 1: Retrieval:** For a given user, query the ScaNN index with their embedding to retrieve a broad set of \~200 relevant candidates. This stage prioritizes recall and speed.

     2. **Stage 2: Re-ranking:** Pass the 200 candidates to a second-stage, lightweight re-ranking model (e.g., **XGBoost** or a small neural network). This model uses a richer feature set, including **cross-features** (direct interactions between user and item features), to precisely score and re-order the candidates.

     3. **Stage 3: Fairness & Diversity Re-ranking:** Apply a final layer of business logic to the ranked list. This includes applying **fairness constraints**, such as boosting scores for items from new or less popular businesses, and ensuring diversity by preventing a single business or category from dominating the top results.

     4. **Stage 4: Filtering & Serving:** Apply final business logic (e.g., filter out-of-stock items) and serve the top N results based on the final, fairness-adjusted scores.

4. **Handling New Products with Continuous Index Rebuilding:**

   * To handle new products with high accuracy and no downtime, implement a "hot-swap" pipeline.

   * When a new product is added, a trigger starts an offline process to generate its true embedding using the trained Product/Business Tower.

   * A new version of the ScaNN index is built incorporating this new embedding.

   * Live servers detect the new index, load it into memory, and atomically switch to it, ensuring new products are available within minutes without impacting live traffic.

5. **A/B Testing & Monitoring:**

   * Roll out the new model to a small percentage of users and compare its business metrics (e.g., click-through rate, conversion rate, average order value) against the previous recommendation system.

   * Continuously monitor model performance and schedule periodic retraining as new data becomes available.

### **Interaction Model Architectures**

#### **User Tower 🙋‍♀️**

* **Objective:** To create a 128-dimension vector representing the user's current context and long-term preferences.

* **Inputs & Processing Layers:**

  * **User ID:** `Input` -> `Embedding Layer` -> **16-dim vector**

  * **Demographics (Location, Age Group):** `Input` -> `Embedding Layer` -> **16-dim vector**

  * **Stated Preferences (Favorite Categories/Brands):** `Input` -> `Embedding Layer` -> **32-dim vector**

  * **Behavioral History (Search Queries):** `Text Input` -> `all-MiniLM-L6-v2` -> **384-dim vector**

  * **Visual Preferences (Aggregated Image Vector):** `Input (1280-dim)` -> `Dense Layer (ReLU)` -> **128-dim vector**

  * **Session/Cart Context (Last N items viewed/added):** `Input` -> `Embedding Layer` -> `Attention Layer/Average` -> **32-dim vector**

* **Interaction & Output:**

  1. All resulting vectors are **concatenated**.

  2. This vector is passed through two `Dense` layers with `ReLU` activation (`1024 -> 512`).

  3. A final `Dense` layer produces the **128-dimension User Output Embedding**.

#### **Product/Business Tower 🍽️**

* **Objective:** To create a 128-dimension vector representing the product's attributes and appeal.

* **Inputs & Processing Layers:**

  * **Product ID:** `Input` -> `Embedding Layer` -> **16-dim vector**

  * **Key Attributes (Category, Price Range, Brand):** `Input` -> `Embedding Layer` -> **32-dim vector**

  * **Description & Review Text:** `Text Input` -> `all-MiniLM-L6-v2` -> **384-dim vector**

  * **Product Image Embedding:** `Input (1280-dim)` -> `Dense Layer (ReLU)` -> **128-dim vector**

* **Interaction & Output:**

  1. All resulting vectors are **concatenated**.

  2. This vector is passed through two `Dense` layers with `ReLU` activation (`1024 -> 512`).

  3. A final `Dense` layer (plus L2 normalization) produces the **128-dimension Product Output Embedding**.
