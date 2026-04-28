
## Zookeeper
 1. What are Zookeeper "other 2 ports" different from 2181 and what are they used for?
 2. What is the use of Zookeeper in Kafka?
 3. Which layer of Broker OCI model does Zookeeper connect to?
 4. What are zNodes? explain Zookeeper leader election?
 
## Kraft
 5. Which is better Kraft or Zookeeper and why?
 6. What are kraft ports? 
 7. Which layer of Broker OCI Model does Kraft connect to?
 8. What happens when there is only one Kraft controller instead of a Quorum?
 
## Kafka Broker

 9. What is IBP(inter broker protocol) and why is it important?
 10. What is log format version and why is it important?
 11. What is unclean leader election and when is it needed?
 12. Whats the difference between ACL vs RBAC?
 13. How do u put all the messages in a specified partition?
 14. Where does Broker perform better Bare metal vs VM?
 15. Which disk is better suited for Broker SSD vs HDD?
 16. If disk is getting full frequently then how would you address this?
 17. What are the mandatory properties to create a topic?
 
## Kafka Producer
 18. What are the producer settings to maximize the throughput?
 19. What are the producer settings to minimize the latency?
 20. What property helps in tackling message ordering?
 21. How do we tackle message duplication?

## Kafka Consumer
 21. What is consumer rebalance? How does it work?
 22. How does Kafka determine if a consumer is dead or doesnt exist?
 23. What is subscribe() vs assign()
 24. What is exactly once semantics how is it different idempotench?
 25. When does Consumer lose data?
 26. When does a consumer get duplication?
 27. What are the consumer settings to maximize the throughput?
 28. What are the consumer settings to minimize the latency?
 29. What property helps in tackling message ordering?

## Kafka Security
 30. What are 5 ACL pillars?
 31. What are listeners and advertised.listeners?
 32. Whats the use of JAAS file?
 33. What are the High level steps to setup SSL in Broker?
 34. What is 2 way and 3 way handshake?
 35. What is session.timeout? How does broker handle client connections?
 36. What is a quota?
 37. How can you increase the client connection duration in Kafka?
 
## Kafka Schema Registry
38. What is schema compatibility? what are different compatibility types? how do we make schema compatible?
39. What is the high level step by step process of introducing a schema to a topic for the first time? 
40. What is schema registration?
41. How do you check schema compatibility?

## Kafka Connect
42. How do you add a worker node to a existing Connect cluster?
43. What is the step by step high level process of installing a new connector?
44. How do you improve a connector performance?
45. What are the mandatory configurations for connect JSON file to create a connector?
46. How does Kafka Connect ensure that workers know about each others configurations? 
47. Where does connect store its committed offset?

## Kafka On-Prem Specific
48. Which one will you choose bare metal vs virtual machine?
49. How is On-Prem capacity planning different from Confluent Cloud Capacity Planning?
50. Which one do you prefer SSD or HDD?
51. What is Zero Copy transfer?
52. How does Kafka client connect to the server at what OCI layer?

 
 

 
  

 
 
 

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI0Nzk5MDE4NV19
-->