## HDFS architecture#
All files stored in HDFS are broken into multiple fixed-size blocks, where each block is 128 megabytes in size by default (configurable on a per-file basis). Each file stored in HDFS consists of two parts: the actual file data and the metadata, i.e., how many block parts the file has, their locations and the total file size, etc. HDFS cluster primarily consists of a NameNode that manages the file system metadata and DataNodes that store the actual data.


  
