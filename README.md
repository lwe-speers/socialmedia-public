# Geberit Social Media Development
- [Project Goal](#project-goal)
- [Project Concept](#project-concept)
- [Python Code](#python-code)
- [SAP Data Intelligence](#sap-data-intelligence)
  * [Social Media Operator](#social-media-operator)
  * [Content of the DI Social Media DOCKERFILE:](#content-of-the-di-social-media-dockerfile-)
  * [Generating the Social Media Operator](#generating-the-social-media-operator)
  * [Using the SocialMedia Operator in a DI Graph](#using-the-socialmedia-operator-in-a-di-graph)
    + [Note:](#note-)
  * [Using the HANA Client in a DI Graph](#using-the-hana-client-in-a-di-graph)
    + [Note:](#note--1)
  * [Steps to update DI Operator from local code repository](#steps-to-update-di-operator-from-local-code-repository)


# Project Goal
Persist Social Media Data into HANA Tables. Thereby the following Social Media Platforms are adressed:
- Twitter
- LinkedIn
- Youtube
- Facebook
- Instagram
- Pinterest

The respective fields are continuously documented within this [file](https://pgag-my.sharepoint.com/:x:/g/personal/lukas_weixler_s-peers_com/EQd-QUM6tbVElBEzMCOvCf4BORq6HV8ewC4xZM8kT8Jz1g?e=MNrbOp)

# Project Concept
![image](https://user-images.githubusercontent.com/81747670/159502749-14319bc3-c40b-4a72-a911-0029f0bec8de.png)

Local Developments are versioned using Git and Github. Production-Ready code is built as a Docker Container and shipped via DockerHub (or Geberit internal Azure) into a SAP Data Intelligence Container. From there, we can use locally developed functionality within a custom operator and implement it into a Data Intelligence Pipeline.

# Python Code

# SAP Data Intelligence
For persisting the retreived Social Media Data we use SAP Data intelligence. to get an introduction to SAP DI, this [Video Series](https://www.youtube.com/playlist?list=PLkzo92owKnVyY89xEshp_cSQ0QF8EE927) is recommended.

## Social Media Operator
The central product of the project is a custom operator, called "Social Media". Locally developed code is shipped through a Docker Container into SAP DI.
Three Tags on the right are used to generate the custom operator in DI
Within DI we retreive the code using a Dockerfile in 'Repository-dockerfiles-SocialMedia':
![image](https://user-images.githubusercontent.com/81747670/177504072-e6bd4f1b-dd90-455f-904a-f4949f32ea73.png)

## Content of the DI Social Media DOCKERFILE:
```
FROM lwxspeers/pyapp:latest

# Install python library "requests"
RUN pip install requests

# Install python library "tornado" (Only required with SAP Data Hub version >= 2.5)
RUN pip install tornado==5.0.2

# Add vflow user and vflow group to prevent error
# container has runAsNonRoot and image will run as root
RUN groupadd -g 1972 vflow
RUN useradd -g 1972 -u 1972 -m vflow

# Change ownership over home folder
RUN chown -R vflow:vflow /home/vflow

# Change user to vflow
USER 1972:1972

# Setting up envs
ENV HOME=/home/vflow
ENV PYTHONPATH=/home/vflow:/home/vflow/relational_engine/src

WORKDIR /home/vflow/relational_engine/src
```

## Generating the Social Media Operator
For using locally developed python functionality in a DI pipeline, have generated a new custom operator, which extends the regular python3 operator. This operator is used within all social media graphs and can be customized for each graph separately.
It is important to set the tags equally as in the above mentioned dockerfile. Thereby, the Operator and Dockerfile can interact.
![image](https://user-images.githubusercontent.com/81747670/177504457-33347b9b-ae59-4109-a3ff-21d5c0c200f8.png)

Within the Script we can now import and use python functionality from our shipped [process.py](https://github.com/s-peers/geberit-socialmedia-development/blob/master/relational_engine/src/process.py) interface:
![image](https://user-images.githubusercontent.com/81747670/177505047-77a229a7-ed27-43dd-b26e-62d41fe7eb43.png)


## Using the SocialMedia Operator in a DI Graph
For each social media platform we use a separate graph. These graphs can be found when searching for 'socialmedia' in the Graphs tab. See the Twitter Graph as an example:
![image](https://user-images.githubusercontent.com/81747670/177506020-073eaa5f-2865-4268-9f61-a81ec4831076.png)

Each Graph is structured around an instance of our DI Operator. Input are API Tokens for the social media platform, output are tables to persist the results from the social media platforms. Within such a model, click on the instance on <> to customize the instance-specific python code:

![image](https://user-images.githubusercontent.com/81747670/177506723-3abd8c53-5330-4434-9423-f69889e8a650.png)

Generally we want as little code as possible within the operator. Therefore in each graph, the operator contains of two functions:
1. `messager()` to bring the data into SAP DI specific message format. This message is then sent to the HANA Operator
2. `main()` which executes the `messager()` when retreiving data from the api inport. This data is structured in table format, therefore we indicate where to find the api token. In the above example we need the token (First row, second column) as well as the Organizational ID (First row, third column). These are used in the `get_twitter_data()` function, which comes from the shipped Docker container, which we have imported into DI. 
### Note:
The `messager()`function will format not all twitter data into a message - just one specific table. For twitter we have two tables: `followerStatistics`and `twitterStatistics`. These are sent to output ports in the operator, which have to be created and named according to the table names:
 rightclick on the operator, add port...
![image](https://user-images.githubusercontent.com/81747670/177509000-2173df47-cb86-476f-ab55-b9a7e8205ad3.png)

## Using the HANA Client in a DI Graph
Use the HANA Client Operator to persist the operator message into a HANA Table. The following configurations are set:
Connection: Configuration Manager, HanaCloud_Dev
Table name: "Schemaname"."Tablename"
Table Columns: They can be either defined in a form or in JSON Format. 
![image](https://user-images.githubusercontent.com/81747670/177510518-0d6be7e2-6533-4d18-afa1-bd4c570bcb51.png)

See the entire configuration for TwitterFollowerStatistics
![image](https://user-images.githubusercontent.com/81747670/177510323-e03b6e06-14de-4748-b2ed-6f6a0cd509ca.png)

### Note:
Beside the graphical interface, the entire SAP DI Logic can also be viewed and edited in JSON Format by clicking the above right icon:
![image](https://user-images.githubusercontent.com/81747670/177511334-260a4f49-e902-406e-82b0-be4680b6c7e3.png)
The entire SAP DI structure is mirrored in this JSON Logic within the DI vscode User Application.
From here I have created a Github repo with a remote. The DI Structure with its JSON files is available [in this repository](https://github.com/s-peers/geberit-di-connection). 

## Steps to update DI Operator from local code repository 
1. Open Terminal in Windows or Pycharm (Alt+F12)
2. Navigate to your code directory
3. Ensure you are on the correct git branch
![image](https://user-images.githubusercontent.com/81747670/178518098-0bd6c3aa-c7cb-4f38-a586-caa9acc9c046.png)
5. Build the Docker Image
docker build -t lwxspeers/pyapp .
![image](https://user-images.githubusercontent.com/81747670/178516239-5265e572-078a-46da-ba16-d80716a9eaff.png)
5. Push the Docker Image to your Repository
docker push lwxspeers/pyapp:latest
![image](https://user-images.githubusercontent.com/81747670/178516528-02ff5da0-8baf-4b29-a396-0daff0155f65.png)
6. Import the Docker Image in DI
- Go to Modeler - Repository - Dockerfiles - SocialMedia - Dockerfile
![image](https://user-images.githubusercontent.com/81747670/178517423-a87ac76e-e8dc-4881-b3ea-583177803602.png)
- Press "Save", Then press "Build", wait until the Image is built succesfully
![image](https://user-images.githubusercontent.com/81747670/178517553-8ac71b52-04cb-44a6-905f-6ea48f4507b1.png)
7. Your updates from local code are now available within the operator. Enter your target DI Model, save and run the model

















