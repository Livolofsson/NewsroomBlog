+++
date = 2025-02-07
title = "Clustering Of Receipts and Market Basket Analysis"
image = "images/dalle.webp"
+++

After extracting my receipts by reverse engineering Kivra's API, it was time for the main mission -to actually get insight from them. Uncover patterns that could help me make better purchase decisions. 

---
## So, did we uncover any patterns? 
**YES!** we did with the help of categorizing items with LLMs, item graphs, Kmeans clustering and market basket analysis to uncover the patterns.

*data analysis*
*graph* 
*clustering*
*market basket analysis* 

---

## Step 1️: Categorize Items with LLM (Large Language Models)  

OpenAIs API was used and the categories was predefined. To not use more tokens than necessary, we're letting gpt-4 categorize the items in batches of 50, that means in each prompt gpt-4 outputs 50 categories (thus inputting 50 rows). The function that prompts the structured output: 

```
    #categories llm has to choose between
    categories = ["fruit", "vegetables", "snacks", "meat", "household", "dairy", "bread", "other"]
    batch_size = 50  

    def generate_prompt(batch_items):
        """
        Generate a prompt for categorizing items.
        """
        item_list = "\n".join([f"{item['primary_key']}: {item['name']}" for item in batch_items])
        prompt = (
            f"You are a categorization assistant. Categorize the following items into one of the predefined categories: "
            f"{', '.join(categories)}.\n\n"
            f"Items:\n{item_list}\n\n"
            f"Respond with the format:\n"
            f"primary_key: category\n\n"
        )
        return prompt
```
For the actual generation of the categories another function is defined that calls OpenAIs API and uses the prompting function to structure the outpus. It returns the categorized items in a list. Some items that involved "Zero" or "julmust" are treated as "snacks" specifically, otherwise she outputed them as "other" but those are definitly in the "snacks" category and especially important in our household as I suspect that we drink a bit too much soda...

```
    def categorize_batch(batch):
        """
        Categorize a batch of items using GPT.
        """
        batch_items = [{"primary_key": int(row["Primary Key"]), "name": row["Item"]} for _, row in batch.iterrows()]
        prompt = generate_prompt(batch_items)
        
        try:
            response = openai.ChatCompletion.create(
                model="gpt-4",  
                messages=[
                    {"role": "system", "content": "You are an assistant that categorizes items into predefined categories. If the item includes 'Zero' or 'julmust' categorize it as snacks."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=500,
                temperature=0
            )
        
            raw_output = response["choices"][0]["message"]["content"].strip()
            print(f"Raw output: {raw_output}") 
            categorized_items = []

            for line in raw_output.split("\n"):
                if ":" in line:
                    primary_key, category = line.split(":", 1)
                    categorized_items.append({
                        "primary_key": int(primary_key.strip()),
                        "category": category.strip()
                    })
            
            return categorized_items
        except Exception as e:
            print(f"Error during categorization: {e}")
            return []
```

In the end the script calls on the categorize_batch() function with the batch size 50 to start the categorization and merge them together with the old csv: 

```
    #processing in batches to reduce cred
    categorized_results = []
    print("Starting categorization process...")
    for i in tqdm(range(0, len(df), batch_size), desc="Processing batches"):
        batch = df.iloc[i:i + batch_size]
        batch_results = categorize_batch(batch)
        if batch_results:
            categorized_results.extend(batch_results)

    #definied categories back to csv
    categorized_df = pd.DataFrame(categorized_results)
    df = df.merge(categorized_df, left_on="Primary Key", right_on="primary_key", how="left")
    df["Category"] = df["category"].combine_first(df["Category"])
    df.drop(columns=["category", "primary_key"], inplace=True)
    
    df.to_csv(updated_csv_file_path, index=False)
    print(f"Updated CSV saved to {updated_csv_file_path}.")
```

Now we have a new column "Category" for each item in the receipt. 

## Step 2️: Exploratory Data Analysis    
Okay, time to look into the data we're working with. This is important because it helps us understand the data, relationships, outliers etc, before doing modeling such as clustering or market analysis. 

After checking if there where duplicates and the type of each column, the following relationship was checked: 

1. Distribution of Purchase Amounts
Prepocessing that had to be done here was to convert "Product Amount" from string with comma (e.g. "14.8") to float (e.g. 14.8)

{{< figure src="distribution_of_purchase_amount.png" title="Postman Debugging" >}}

The majority of purchases were small, typically under 50 SEK, indicating that most transactions are quick, low-cost items rather than full grocery hauls. However, there is a long tail of larger purchases, suggesting occasional bulk or special-purpose shopping.

2. Shopping Hours Distribution
Extracted the hour from "Purchase Date" (e.g. "2024-11-29, 21:05") using 
```
data["Hour"] = data["Datetime"].dt.hour
```

{{< figure src="shopping_hour_distribution.png" title="Postman Debugging" >}}

Most shopping occurred between 18:00 and 20:00, showing a strong evening peak. This likely reflects after-work shopping behavior.

3. Average spending per Day of the Week
Converted date to weekday name using
```
data["DayOfWeek"] = data["Datetime"].dt.day_name()
```

{{< figure src="average_spending_per_day_of_the_week.png" title="Postman Debugging" >}}

Spending was highest on Sunday, Monday, and Friday, suggesting a behavioral pattern. It could be prepare for the week ahead (Monday), stock up for the weekend (Friday), or do larger shops at the end of the week (Sunday). However the peaks are very small.

4. Total spending by Store

Grouped by "Store Name" and summed the "Product Amount Numeric". It didn't provide much insight as all purchases was made from one store: 
```
store_spending = raw_clustering_df.groupby("Store Name")["Product Amount Numeric"].sum()
```

5. Number of Purchases by Category

{{< figure src="number_of_purchases_by_category.png" title="Postman Debugging" >}}

Snacks, dairy and vegetables were the most frequently purchased categories. These appear to be everyday staples or/and impulse buys.

6. Weekend vs Weekday Analysis
Used "DayOfWeek" to determine IsWeekend and classified it as 1 or 0. The binary encoding was stored in a new column. 

```
data["IsWeekend"] = data["DayOfWeek"].isin(["Saturday", "Sunday"]).astype(int)
```

{{< figure src="total_spending_weekends_vs_weekdays.png" title="Postman Debugging" >}}

{{< figure src="average_spending_per_transaction.png" title="Postman Debugging" >}}

On weekends, average spending per transaction was higher, but the total spending was higher during weekdays. This suggests that we make larger but fewer purchases on weekends and more frequent smaller purchases during the week.

7. Feature Correlation Matrix: Between numeric features like hour, day, amount, etc. 

Selected only numeric columns such as Hour, Day, IsWeekend, Product Amount Numeric

```
numeric_df = raw_clustering_df.select_dtypes(include=['number'])
```

{{< figure src="feature_correlation.png" title="Postman Debugging" >}}

No strong correlations were found between numeric features such as hour, day, weekend flag, or amount spent. This indicates that each feature adds independent value, making them all useful inputs for clustering without redundancy.


## Step 3️: Clustering 
Clustering is an unsupervised learning technique (which means that we do not know the classes each data point, receipt in this case, belong to from the beginning) that finds natural groupings, thus patterns, in the data. 

In this case, I'd be nice to uncover behaviour patterns in myself so that I know what I should strive to (or avoid) to make better purchases, by means saving cash. Patterns could be, snack shopping in the evening, then maybe these purchases could be avoided by trying to make majority of purchases during day... because then the snack cost might decrease, etc. 

Some preprocessing of the data had to be done:

1. Day of week and month went from dates to cosine, sin values to make them numerical and capture there cyclic nature. 
2. Time was translated to only hour and cosine and sin was used to translate hour to values that capture the cylic nature here as well. 
3. Product Amount was normalized using StandardScaler()

The Elbow Method is a way to figure out how many clusters that are optimal for the data when the clustering will be done with KMeans. This is done by plotting "inertia" (within-cluster-error) against number of clusters. Where the change drops fastest, is the optimal number of clusters.

{{< figure src="elbow.png" title="Postman Debugging" >}}

In this case the optimal number of clusters is four. Kmeans will now group similar data points into k = 4 clusters for this data. 

```python
####elbow method#####
file_path = "C:/Users/astri/KivraReceipt/clustering_data_cyclical_onehot.csv"  # Adjust if needed
data_to_cluster = pd.read_csv(file_path)

#Converting all columns to float (to ensure correct numerical processing)
features = data_to_cluster.astype(float)

inertia = []
K_range = range(1, 11) 

for k in K_range:
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    kmeans.fit(features)
    inertia.append(kmeans.inertia_)

plt.figure(figsize=(8, 5))
plt.plot(K_range, inertia, marker='o', linestyle='-', color='blue')
plt.xlabel("Number of Clusters (k)")
plt.ylabel("Inertia (Sum of Squared Distances)")
plt.title("Elbow Method for Optimal k")
plt.xticks(K_range)
plt.grid()
plt.show()

###########K-means with k=4################
kmeans = KMeans(n_clusters=4, random_state=42, n_init=10)
data_to_cluster["Cluster"] = kmeans.fit_predict(features)
centroids = pd.DataFrame(kmeans.cluster_centers_, columns=features.columns)

plt.figure(figsize=(6,5))
data_to_cluster["Cluster"].value_counts().sort_index().plot(kind="bar", color="blue", edgecolor="black")
plt.xlabel("Cluster")
plt.ylabel("Number of Transactions")
plt.title("Number of Transactions Per Cluster")
plt.show()

#reducing to 2 components for visualization (corr matrix showed not much correlation so PCA probably didn't help much)
pca = PCA(n_components=2)
data_pca = pca.fit_transform(features)

plt.figure(figsize=(8,6))
plt.scatter(data_pca[:, 0], data_pca[:, 1], c=data_to_cluster["Cluster"], cmap="viridis", alpha=0.6)
plt.xlabel("PCA Component 1")
plt.ylabel("PCA Component 2")
plt.title("Clusters Visualized in 2D Space (PCA)")
plt.colorbar(label="Cluster")
plt.show()
```

PCA (Principal Component Analysis) is a way to vizualize the clusters by reducing data with many dimensions, where a column is countet as one dimension, to less dimensions. The result was the following 

{{< figure src="clusters.png" title="Postman Debugging" >}}

However, looking at the correlation matrix from before, features were mostly uncorrelated. As PCA works by combining features to one by using the variance, if they don't have the same variance, the combination to get new features might not work very well. 

We can se some clusters do, even though they're overlapping. These could be analyzed further to see which features that are highly correlated in the clusters to understand purchase behaviour.

## Step 4️: Market Basket Analysis   

{{< figure src="market.png" title="Postman Debugging" >}}
---


## Step 5: Graph Items   

{{< figure src="graph.png" title="Postman Debugging" >}}
---