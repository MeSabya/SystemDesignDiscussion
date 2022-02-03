# Count Min Sketch 

it is a probabilistic data structure that takes **sub-linear space** to store the number of occurrences of items in a stream. 
One trade-off of this data structure is that sometimes it over-counts these occurrences.

👉 *Count-min sketch uses a matrix of counters to store multiple hash functions. In this matrix, the rows indicate the hash functions and columns is the range of hash codes produced by these hash functions.*

![image](https://user-images.githubusercontent.com/33947539/152286492-51d19d8f-6fed-46e7-9b5c-a2b1e7c1d8c4.png)

In the above diagram, we have four different hash functions H1, H2, H3, H4 represented by four different rows. The number of columns is 10 which indicates that every hash function in our algorithm will produce hash codes in the range 1–10. If we have a continuous stream of incoming data, we will calculate hash codes for each item and increment the respective counter in our matrix.


![image](https://user-images.githubusercontent.com/33947539/152286528-47212d5e-9a40-430e-959a-2f45df9840cf.png)

☝️ A sample input stream

**Step1**: For simplicity, consider a stream of incoming characters as shown above. Let us begin with the first character A, feed this A to the four hash functions.

![image](https://user-images.githubusercontent.com/33947539/152286682-5d3acefd-ecfa-49eb-a941-f129c5424cff.png)

**Step2**:  Now increment the counters for these values in our matrix. For example, H1 gives 1 so go to row 1 and increment the value o column 1. Similarly, for H2 we get 3, go to row 2 and increment the value of column 3. For H3 increment the value in column 8 and finally for H4 increment the value in column 1. These are circled in the image below for better understanding:

![image](https://user-images.githubusercontent.com/33947539/152286716-2c484595-1b46-4da1-8d27-e2f8e20dc06d.png)

**Step3**: Similarly, let us calculate the hash code for the next element in our stream, B

![image](https://user-images.githubusercontent.com/33947539/152286939-4a2148dc-ce43-4d53-97f2-12eafd0af717.png)

and increment the counters for these values. Notice how the count for row 4, column 1 is now 2. It was previously 1 but the value is incremented after calculating the hash code for B

Repeat the above step-2, 3 for each character in the input stream. Repeat this for every element in the stream (A, B, C, L) and increment the counter every time, the final matrix looks like:

![image](https://user-images.githubusercontent.com/33947539/152287039-f8d4228d-eea3-4fb0-8bff-978215f586c7.png)

**Step4**:
Using this matrix, we can calculate the number of occurrences of A, by finding the minimum of all the counters in our sketch/matrix.

Step 1: calculate the hash codes for A : H1: 1, H2: 3, H3: 8, H4:1

Step 2: Fetch the values for these counters from our sketch (these are marked in bold in our final sketch): 5, 4, 4, 7

Step 3: Find the minimum value in the counter: 4. Tadaaa…. This means A occurs 4 times in our stream.


## Usecases:
Some common applications of this amazing data structure are listed below:

1. To find frequent items sold for an online retailer: the product sold can be the item and frequency can the number of times the item was sold.

2. To make a password strong, we can keep track of how many times a password was used for a given website. In this scenario, the password is the item and frequency is the number of times the given string was used as a password.

3. Database Query Plan: Databases engines use the count-min sketch to find good execution strategies. If we have a query like select * from a, b where a.n = b.n, to estimate the size of the join we can maintain two sketches, one for occurrences of n in a and other for n in b. We can then query these sketches to estimate the size of the join.

## References
https://github.com/21zhouyun/CountMinSketch/blob/master/main.py


