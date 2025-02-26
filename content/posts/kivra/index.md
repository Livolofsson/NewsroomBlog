+++
date = 2025-02-07
title = "Reverse Engineering Kivra's API"
image = "images/kivratest2.png"
+++


Kivra is a digital service that many companies use to send documents to Swedish citizens, such as bills, tax notices and an occasional reminder such as “Hey, it’s time for a doctor visit!”  letting you know you’re not invincible anymore.  

It also collects all receipts of purchases you’ve made, if you want them to! The coolest part? You automatically stop receiving paper documents in your mailbox  or receipts in-store, because you’ve signed up to get them digitally instead! It’s the perfect gift for adulthood. 😉

---

## If you’re like me, you’ve probably wondered: 
- **Can I uncover patterns/trends in my purchase history to make better financial decisions?**
- **Am I a responsible adult or just really good at pretending? 🤔**
- **Can I track my receipts like some kind of financial detective?**

The answer is **YES!**  But here’s the twist—Kivra doesn’t exactly hand over this data in a neat little export button. So, like any curious soul (or mildly obsessive person), I reverse-engineered my way into grabbing my own receipts programmatically. 

---

## Step 1️: Break into—Wait, No, Just Log Into Kivra   
Before we start our (totally legal and ethical) data heist, you’ll need to log into Kivra using BankID. A digital idenfication system widely used in Sweden.

This post, however, is all about how to actually get access to your receipts as a private person from Kivra. If documentation existed about how to acquire that as a private person, then this post wouldn't have existed. 😅 We'll have to reverse engineer their API.

Once logged in with your BankID, it’s pretty straightforward though. Don’t worry, this is not hacking—this is just playing Sherlock Holmes. 😎 Reverse engineering is like hacking but having the law on your side.

---

## Step 2️: Spy on the Network   
Okay, time to do what every nerd loves: looking underneath the hood.

1. Right-click to open “Inspect”.
2. Click on the **Network tab**. 
3. Refresh the page and watch Kivra’s API spill all its secrets right in front of you. 

You’ll see a flood of network requests, but we’re looking for one specific operation—something like receipt.

{{< figure src="behindhood.png" title="Inspecting Network Requests" >}}

---

## Step 3️: Decode the API Calls with Postman  
_(Optional, but super helpful!)_  

If you’re already comfortable with reverse-engineering APIs, you might skip this. Otherwise, **Postman** helps figure out the puzzle pieces of our detective work. 

1. Paste the request into Postman and click **Send**.  
2. You’ll get error messages telling you what’s missing.  
3. Error messages are our best friends—they guide us to missing parameters.  
4. Just go back to **Network Inspect Mode**, grab the missing pieces, and try again.  

{{< figure src="errorpostman.png" title="Postman Debugging" >}}

Two API requests are needed:  
- One with the **operation name `receipt`**, found in the overview page in Kivra. When you've reconstructed the request successfully in postman you'll get a json body looking like this: 

```
{
    "data": {
        "receiptsV2": {
            "__typename": "ReceiptListV2",
            "total": 243,
            "offset": 0,
            "limit": 1,
            "list": [
                {
                    "__typename": "ReceiptBaseDetails",
                    "key": "502dfbf36f4ffdcba10cb78851c1dcf9f1366d98063",
                    "purchaseDate": "2025-02-13T16:44:39Z",
                    "totalAmount": {
                        "formatted": "75,25 kr"
                    },
                    "store": {
                        "name": "ICA",
                        "logo": {
                            "publicUrl": "https://static.kivra.com/receipt/158410061928d4a9a251d546dba34a6361e37e7537"
                        }
                    },
                    "accessInfo": {
                        "owner": {
                            "isMe": true,
                            "name": "Sam Altman"
                        }
                    }
                }
            ]
        }
    }
}

```
- Another with **receipt details**, found by checking the API requests while opening a specific receipt. A successfull request in Postman will have a respone comes in json format. A subset of the body looks like this: 

```
{
    "data": {
        "receiptV2": {
            "key": "2502dfbf36f4ffdcba10cb78851c1dcf9f1366d98063",
            "content": {
                "header": {
                    "totalPurchaseAmount": "75,25 kr",
                    "subAmounts": [
                        "Poänggrundande belopp 75,25 kr"
                    ],
                    "isoDate": "2025-02-13T16:44:39Z",
                    "formattedDate": "2025-02-13, 17:44",
                    "labels": [],
                    "logo": {
                        "publicUrl": "https://static.kivra.com/receipt/1579618096bac60a57676c4f78be1c6af5025d513a"
                    }
                },
                "items": {
                    "allItems": {
                        "text": "Produkter",
                        "items": [
                            {
                                "text": [],
                                "type": "product",
                                "name": "Apelsin",
                                "money": {
                                    "formatted": "22,00 kr"
                                },
                                "quantityCost": null,
                                "deposits": [],
                                "costModifiers": [],
                                "identifiers": []
                            },
                            {
                                "text": [],
                                "type": "product",
                                "name": "Exotic Zero",
                                "money": {
                                    "formatted": "24,95 kr"
                                },
                                "quantityCost": null,
                                "deposits": [
                                    {
                                        "description": "Pant",
                                        "money": {
                                            "formatted": "2,00 kr"
                                        },
                                        "isRefund": false
                                    }
                                ],
                                "costModifiers": [],
                                "identifiers": []
                            },
                            {
                                "text": [],
                                "type": "product",
                                "name": "Lösviktsgodis",
                                "money": {
                                    "formatted": "22,35 kr"
                                },
                                "quantityCost": {
                                    "formatted": "0,205 kg * 109,00 kr/kg"
                                },
                                "deposits": [],
                                "costModifiers": [],
                                "identifiers": []
                            },
                            {
                                "text": [],
                                "type": "product",
                                "name": "Plastkasse",
                                "money": {
                                    "formatted": "3,95 kr"
                                },
                                "quantityCost": null,
                                "deposits": [],
                                "costModifiers": [],
                                "identifiers": []
                            }
                        ]
                    }
                }
            },
            "sender": {
                "name": "ICA",
                "key": "15765665678a499fdd66139b23016615a978111111"
            },
            "attributes": {
                "isUpdatedWithReturns": false
            }
        }
    }
}
```

What's important to note here is that by getting the **key** in the first response, we can use it to get receipt detail by inserting the **key** in our receipt detail request. This is useful for our Python script later as it enables us to fetch details about many receipts automatically.

---

## Step 4️: Automate with Python   
Once I knew these API calls worked in Postman, it was time to write a Python script to fetch all receipts using the `requests` library. 

**What I did:**  
1. **Requested all unique keys** that define each receipt.  
```python
import requests
from dotenv import load_dotenv
import os 
import csv 
import pandas as pd
from tqdm import tqdm
from datetime import datetime 

load_dotenv()
token = os.getenv("KIVRA_AUTH_TOKEN")
x_actor_key = os.getenv("x-actor-key")

url = "https://bff.kivra.com/graphql"
list_receipt_body = {
    "operationName": "Receipts",
    "query": "query Receipts($search: String, $limit: Int, $offset: Int) {\n  receiptsV2(search: $search, limit: $limit, offset: $offset) {\n    __typename\n    total\n    offset\n    limit\n    list {\n      ...baseDetailsFields\n    }\n  }\n}\n\nfragment baseDetailsFields on ReceiptBaseDetails {\n  key\n  purchaseDate\n  totalAmount {\n    formatted\n  }\n  attributes {\n    isCopy\n    isExpensed\n    isReturn\n    isTrashed\n  }\n  store {\n    name\n    logo {\n      publicUrl\n    }\n  }\n  tags {\n    key\n    name\n    icon\n  }\n}",
    "variables": {
        "limit": 1000,
        "offset": 0,
        "search": None
    }
}
headers = {
    "x-actor-key": f"{x_actor_key}",
    "Authorization": f"Bearer {token}"
}

print("Started fetching receipts")

response = requests.post(url, json=list_receipt_body, headers=headers)
result = response.json()

receipt_keys = [dicti["key"] for dicti in result["data"]["receiptsV2"]["list"]]
print(f"Fetched {len(receipt_keys)} of {result['data']['receiptsV2']['total']}")

```
2. Fetched the details of all receipts.   
3. Saved the data in a structured format. In my case it was a CSV, with details such as the ones seen in csv_headers in the code below.

```py3
csv_headers = ["Primary Key", "Receipt Key", "Purchase Date", "Store Name", "Item", "Product Amount", "Category", "Product Amount Numeric"]
csv_file_path = f"receipts_{datetime.now().strftime('%Y_%m_%d')}.csv"

primary_key_count = 1

with open(csv_file_path, mode="w", encoding="utf-8-sig", newline="") as f: 
    writer = csv.DictWriter(f, fieldnames=csv_headers)
    writer.writeheader()
    for key in tqdm(receipt_keys, desc="Processing receipts"): 
        receipt_details_body = {
            "operationName": "ReceiptDetails",
            "query": "query ReceiptDetails($key: String!) {\n  receiptV2(key: $key) {\n    key\n    content {\n      header {\n        totalPurchaseAmount\n        subAmounts\n        isoDate\n        formattedDate\n        text\n        labels {\n          type\n          text\n        }\n        logo {\n          publicUrl\n        }\n      }\n      footer {\n        text\n      }\n      items {\n        allItems {\n          text\n          items {\n            text\n            type\n            ... on ProductListItem {\n              ...productFields\n            }\n            ... on GeneralDepositListItem {\n              money {\n                formatted\n              }\n              isRefund\n              description\n              text\n            }\n            ... on GeneralDiscountListItem {\n              money {\n                formatted\n              }\n              isRefund\n              text\n            }\n          }\n        }\n        noBonusItems {\n          text\n          items {\n            type\n            ... on ProductListItem {\n              ...productFields\n            }\n          }\n        }\n        returnedItems {\n          text\n          items {\n            type\n            ... on ProductReturnListItem {\n              name\n              money {\n                formatted\n              }\n              quantityCost {\n                formatted\n              }\n              deposits {\n                description\n                money {\n                  formatted\n                }\n                isRefund\n              }\n              costModifiers {\n                description\n                money {\n                  formatted\n                }\n                isRefund\n              }\n              connectedReceipt {\n                receiptKey\n                description\n                isParentReceipt\n              }\n              identifiers\n              text\n            }\n          }\n        }\n      }\n      storeInformation {\n        text\n        storeInformation {\n          property\n          value\n          subRows {\n            property\n            value\n          }\n        }\n      }\n      paymentInformation {\n        text\n        totals {\n          text\n          totals {\n            property\n            value\n            subRows {\n              property\n              value\n            }\n          }\n        }\n        paymentMethods {\n          text\n          methods {\n            type\n            information {\n              property\n              value\n              subRows {\n                property\n                value\n              }\n            }\n          }\n        }\n        customer {\n          text\n          customer {\n            property\n            value\n            subRows {\n              property\n              value\n            }\n          }\n        }\n        cashRegister {\n          text\n          cashRegister {\n            property\n            value\n            subRows {\n              property\n              value\n            }\n          }\n        }\n      }\n    }\n    campaigns {\n      image {\n        publicUrl\n      }\n      title\n      key\n      height\n      width\n      destinationUrl\n    }\n    sender {\n      name\n      key\n    }\n    attributes {\n      isUpdatedWithReturns\n    }\n  }\n}\n\nfragment productFields on ProductListItem {\n  name\n  money {\n    formatted\n  }\n  quantityCost {\n    formatted\n  }\n  deposits {\n    description\n    money {\n      formatted\n    }\n    isRefund\n  }\n  costModifiers {\n    description\n    money {\n      formatted\n    }\n    isRefund\n  }\n  identifiers\n  text\n}",
            "variables": {
                "key": f"{key}"
            }
        }
        response = requests.post(url, json=receipt_details_body, headers=headers)
        result = response.json()
        receipt = result["data"]["receiptV2"]["content"]

        items = receipt["items"]["allItems"]["items"]
        store_name = receipt["storeInformation"]["storeInformation"][0]["property"]
        purchase_date = receipt["header"]["formattedDate"]
        
        for item in items:
            if "name" not in item:
                continue
            name = item["name"]
            product_amount = item["money"]["formatted"]
            product_amount_numeric = float(item["money"]["formatted"].replace(",", ".").replace("kr", ""))
            writer.writerow({
                "Primary Key": f"{primary_key_count}",
                "Receipt Key": f"{key}", 
                "Purchase Date": f"{purchase_date}",    
                "Store Name": f"{store_name}", 
                "Item": f"{name}", 
                "Product Amount": f"{product_amount}", 
                "Category": "None", 
                "Product Amount Numeric": f"{product_amount_numeric}"
            })
            primary_key_count += 1

df = pd.read_csv(csv_file_path, encoding="utf-8-sig")

```  

Now, I have a wonderful little CSV file that I can analyze to find purchase patterns! 🤩

---