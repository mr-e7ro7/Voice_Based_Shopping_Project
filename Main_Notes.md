# Links
- [Agentic workflow](https://github.com/mr-e7ro7/Voice_Based_Shopping_Project/blob/main/Embedded_Notes/Agentic_Workflow.md) 
- [RAG and Embeddings](https://github.com/mr-e7ro7/Voice_Based_Shopping_Project/blob/main/Embedded_Notes/RAG_and_Embeddings.md)
- [Qdrant or Chroma ](https://github.com/mr-e7ro7/Voice_Based_Shopping_Project/blob/main/Embedded_Notes/Qdrant_and_Chroma.md)
	-  [Chroma Docs](https://docs.trychroma.com/docs/overview/introduction)
	* [Chroma GitHub](https://docs.trychroma.com/docs/overview/introduction)
* [Vector store](https://github.com/mr-e7ro7/Voice_Based_Shopping_Project/blob/main/Embedded_Notes/Vector_Stores.md) 
	* [Difference between Embeddings and Vector stores](Summer_Project/Embedded_Notes/Embeddings_vs_Vector_Stores) 
* LiveKit
	* [LiveKit Docs](https://docs.livekit.io/intro/overview/)

# Architecture


```
——————————————————————————————————————————————————
|                   LiveKit                      |
|    STT and TTS for communication with LLM      |
——————————————————————————————————————————————————
      |                          |
      |                          | Fetch customer data using phone number
      |                          | to query Customer_Info
      V                          V
————————————————        ——————————————————
|     LLM      |        |    Customer    |
|   (Gemini    | ◄────— |     Data       |
|    API)      |        ——————————————————
————————————————
      ^
      |
      | Tool calls
      | (search, stock, cart)
      |
      V
——————————————————————————————————————————————————
|                   Database                     |
|                                                |
|  ———————————————   —————————————————————————   |
|  |   ChromaDB  |   |    PostgreSQL         |   |
|  | • Product   |   | • Product_Info        |   |
|  | • Customer  |   | • Customer_Info       |   |
|  |   History   |   | • Deals               |   |
|  ———————————————   | • Cart/Session        |   |
|                    —————————————————————————   |
——————————————————————————————————————————————————
      |
      | On cart confirmed
      V
——————————————————————————
|    Payment Dispatch    |
|                        |
| Generate link:         |
|   (Razorpay/Cashfree)  |
|                        |
| Send via:              |
|   WhatsApp (Twilio)    |
——————————————————————————
```



## LiveKit

- Handles the speech-to-text, and text-to-speech for the LLM
- LiveKit Room manages the WebRTC session per incoming call
- LiveKit Agent Worker is the Python/Node process that connects to the room, runs your LLM logic, and speaks back
- Each concurrent call gets its own LiveKit Room + Agent Worker instance. 

## DataBase

### Chroma

1. **Product_Search**
	- Stores information about a product relevant for querying, such as name, description, and categories. 
	- Also contains the product id for easy retrieval of product details
	
2. Customer_History
	- Stores the purchase history of the customer.


### SQL DataBase

1. **Product_Information** 
	- Used to store product details and manage inventory.

2. **Customer_Information**
	- Used to store the information of customers, like name, email, e.t.c.

3. Deals
	- Used to store the available offers.

4. Cart
	- Weak relation with Customer_Information as the owner

