## What is a search system?#

***A search system is a system that takes some text input, a search query, from the user and returns the relevant content in a few seconds or less. 
There are three main components of a search system, namely:***

- A crawler, which fetches content and creates documents.
- An indexer, which builds a searchable index.
- A searcher, which responds to search queries by running the search query on the index created by the indexer.

## Requirements

### Functional requirements:
The following is a functional requirement of a distributed search system:

**Search:** 
Users should get relevant content based on their search queries.

### Non-functional requirements:
Here are the non-functional requirements of a distributed search system:

- **Availability**: The system should be highly available to the users.
- **Scalability**: The system should have the ability to scale with the increasing amount of data. In other words, it should be able to index a large amount of data.
- **Fast search on big data**: The user should get the results quickly, no matter how much content they are searching.
- **Reduced cost**: The overall cost of building a search system should be less.

## Resource estimation
### Number of servers estimation
![image](https://user-images.githubusercontent.com/33947539/199938622-c86ef523-77aa-4111-93da-44e6d610650e.png)

### Storage estimation
![image](https://user-images.githubusercontent.com/33947539/199941724-1dbfbd59-95cc-48ec-ab4a-7e9fd44caa7b.png)

### Bandwidth estimation
![image](https://user-images.githubusercontent.com/33947539/199942325-63f3f871-60a5-4a4d-a479-d4c423fd2d6c.png)


 
