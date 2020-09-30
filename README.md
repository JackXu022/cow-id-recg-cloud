# CS5412 Project
Boteng Yao, Zixiao Wang, Zhaopeng Xu

# 1. Introduction
<div style="text-align: justify">
The goal of our project is to recognize cow identity on the farm based on the pattern of their back. We leveraged Azure Cloud and edge computing to identify the cow ID. Our Cow Recognition Platform provides single image prediction service and real time video processing service. The system is separated by the cloud computing part and the edge computing part. Our platform can serve farmers from all over the world. We can also provide customized service based on the farmer’s demand. 
</div>

# 2. System Design
This Chapter will mainly discuss the design of the real time video process service and global single image prediction service.

## 2. 1 Real Time Video Service
The following picture shows the real time video process design. The system is splitted by two parts, one is cloud computing and the other is edge computing. The cloud will do the complicated deep learning job to predict the cow ID, and the edge computing will sample the video to images, resize images and predict if there is a cow. Then, the resized cow image will upload to the Azure Blob, in the meantime, a message is written to the message queue.  For the message queue, there are two implementations. One is all farms will send the messages to a single queue, and a Function will distribute these messages to the specific prediction Functions. The other approach is each farm is assigned with one queue. Customers can choose their services based on demand.

![image](https://github.com/calmbryan/cow-id-recg-cloud/blob/master/img/functions.jpg?raw=true)

### Cloud Computing Design

**Keywords**: Azure Message Queue; Azure Functions; SignalR; Azure Blob Storage;


For the cloud part, Azure Message Queue, Azure Functions, Azure SignalR and Azure Blob are used. Edge server will upload one modified image and send one message including the image urls and farm id to the message queue. The consumers of the queue are Azure Functions, and they will auto scale based on the number of messages in the queue. Azure Function will fetch the image data from blob storage by URL and then it will begin to predict the Cow ID.  One thing to mention here, the container in these functions has finished the loading of a deep learning model. These will decrease the overhead time to run a function. When Azure Functions finish the prediction job, the results will be sent to the corresponding farm through SignalR service by websocket. In order to establish the  connection between SignalR service, the farm edge server needs to communicate with a negotiation function to get an identified key. 

Each farm will be assigned an Azure Blob Storage account. And the image data will be splitted into different shards. Azure Blob Storage is a cloud file system and it also provides high scalability, consistency and availability.

### Edge Computing Design
**Keywords**: Video Sampling; Edge detection;  Image Storage; Threading Pool; MySQL cluster;

Edge computing design consists of four parts of functionalities: video sampling, cow existence detection, data-uploading to cloud and image-storage. 

First, the edge computing system would take a piece of video as input, which could be obtained from both camera or local file system. The video would be sampled by OpenCV VideoCapture in order of sequence. 

Second, the captured image would be passed to a machine learning model to detect if there’s a cow in the frame, giving a binary output as the result. Because given most of the time there won’t be any cow in the frame, it’s a waste of resources to process all of the frames. The machine learning model is pre-trained and exported from Microsoft Custom Vision. The TensorFlow dependency and the pb file is located on edge. The empty image would be blocked and only images with cows would be passed to the process following.

Third, once we make sure the cow is in the frame, the edge would upload the image to Azure Blob Storage. After the completion of the uploading, it would write the Blob Storage URL as a message to the Azure Message Queue to further trigger the Azure functions on cloud. This task is implemented in the fashion of threading pool for acceleration. Each image would be taken care of by one of the 20 threads doing the same upload task.

Finally, when the recognizing work is done and results are sent back from the cloud, they would be written in MySQL on edge, each with URL and a predicted label. MySQL on edge is implemented in the fashion of master-slave architecture and built with docker.  Master and slave servers were set up, and they will communicate with each other by TCP to keep consistency. Any changes made to the write master are synchronized to the read replicas. Therefore, there is replica lag, but the time can be tolerated in a single farm database because read operation doesn’t have an effect on the system. In this case, the most important object for the slave is to avoid data loss rather than to reduce the load pressure.

## 2.2 Single Image Prediction Service
For single image prediction demands, we also implemented a global service image prediction platform which is built on Azure Kubernetes Service. Customers can upload the image from the client side to the cluster and get the prediction results. The image will go first to the first tier service written by Go, then the service will determine which model for the image should be used. Then the results will be returned to the client. The following picture shows the whole system architecture.

![image](https://github.com/calmbryan/cow-id-recg-cloud/blob/master/img/kubernetes.jpg?raw=true)

The first tier consists of four parts of functionalities: hash and upload images, task distribution, update or insert metadata to MySQL cluster, and communication with Redis cache system. 
First, images with farm IDs are uploaded to the first tier service. The Go program will use SHA1 hash function to hash the image, and it will upload images to different image storage shards based on hash value and farm ID. Second, the first tire will distribute the prediction tasks to different prediction python prediction instances according to the farm ID. 

Then, the returned prediction results with image URL are inserted to MySQL database and in the meantime, these metadatas are also inserted to Redis cache. That means not all the images will be uploaded to image storages. If an image is predicted before, the Redis cache will return the results directly. This will significantly reduce the load pressure of the whole system. Cache system is very useful to reduce the pressure of the system just like the Facebook S4LRU system.

Besides, how about the situation if Redis doesn't cache the results but the image has been uploaded before? The answer is to query the hash key from the MySQL database. A master-slave and write/read splitted MySQL database cluster is built. From the system’s point of view, the query will only go to the read replica of MySQL, which will also reduce the database  pressure.  The replica lag is not a big issue for our system because the system will continue to predict and update the database.

The user interface is like:

![image](https://github.com/calmbryan/cow-id-recg-cloud/blob/master/img/frontend.jpg?raw=true)
