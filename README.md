# Automating Security Analyst Activities   
In every SOC environment, there's a constant need for development a better use cases, automation, strategies and many other aspects.

Being able to automate tasks can provide a Security Analyst with much more time to be able to investigate in more cases. Here comes the SOC Engineer in the rescue to enable the Analyst to do much more.

I've put some pieces together to enable a simple task such as manually investigating in a range of IP addresses in the sense that one of them may seem malicious, we can automate this task easily via a set of **APIs** and the lovely **Tines** platform.

I'm using **IBM QRadar** as my main SIEM solution in which the Security Analyst uses to investigate further in cases, also, I'm going to use **VirusTotal** as my main crowdsourcing platform to check whether this IP address was malicious or not.

## Setup  Algorithm 
 ![System Overview](https://i.imgur.com/h8tt245.png)

In this diagram you can see that the communication is done via **APIs**, we are going to automate this whole process via **Tines** platform. To clarify this we can surely do this via a simple Python script.

The whole algorithm can be summarized into main bullet points

 - A list of IP addresses are grabbed from the SIEM solution
 - This list of IP addresses is then passed to the VirusTotal's API to check whether this IP address is malicious or not
 - If in fact, the IP address was malicious, Tines would fire an alert to a specified email address warning them of a potential intrusion.


## Actual Setup 
Firstly, we need to get either an authorization service token, or HTTP basic authorization that are going to be hard-coded to base64 encoding.

We can get the authorization service token in QRadar, in the Admin tab -> Authorized Services.
![qradar-admin](https://i.imgur.com/fXNt5zk.png)

After choosing that option we can authorize a token which later we can use to make API calls.
![token-service](https://i.imgur.com/3UrpjJs.png)

![token-created](https://i.imgur.com/lDoTkUl.png)
---
After successfully obtaining our token, we can then develop our grabbing algorithm, we can do a lot of things to grab the IP address successfully, we are choosing to grab the IP address via AQL query in which we can utilize the QRadar's Ariel database to grab the IP address.

Following the [IBM QRadar API catalouge](https://www.ibm.com/docs/en/qradar-on-cloud?topic=api-accessing-interactive-documentation-page), we can successfully know exactly which part we're using, which in this case ***/api/ariel***.

We can access the API Documentation in our SIEM environment via ***[your-siem-ip]/api_doc***, in there we can find many options we can use to utilize API calls, as we've said before we're only interested in the ***ariel database*** part only. 

![api-doc](https://i.imgur.com/SZPHxFh.png)

We can see in the left pane **searches**, **search_id** and **results**. In fact, we're only going to use **searches** and **results**.

**Searches** is going to benefit us via getting the **search_id** which we can use later on in the **results** tab to retrieve our query results.

The query is going to depend on what scenario you are trying to deal with, in our case we think that there might be a persistent connection between one of our devices and a malicious destination IP adderss, so we were tasked to find out which IP address of those is malicious or not.

The query in this example

```sql
SELECT destinationip FROM flows START '2024-09-08 08:05:00' STOP '2024-09-08 08:10:00'
```
We're trying to grab all of the destination IP addresses in a specific range of time we think that the connection might have been established.

If we actually execute this query we see that a set of IP addresses are retrieved with duplicates which can indicate that there might be some jittering techniques that were used
![query-exec](https://i.imgur.com/34BJUo7.png)

Back to our API documentation, when we apply our query to POST method in the **searches** tab we can see that the **search_id** was retrieved

![query-exec2](https://i.imgur.com/dna4n3Z.png)

![search-id](https://i.imgur.com/ugcGmOV.png)

Now, we go to **results** tab and pass that **search_id**.

![search-id-exec](https://i.imgur.com/VccjUtU.png)

Here, we can see the results of our **search_id**.

![results-1](https://i.imgur.com/zrwvWZK.png)

***NOTE! All of those API calls were executed on the API documentation environment of the SIEM solution***

## TIME TO AUTOMATE ALL OF THIS!!

As said before we're using **Tines** platform to automate our whole process and as we've said before this can be automated via a bash script or a python script too. 

Mainly this isn't a **Tines** guide, but a quick introduction is needed. Tines is literally an automation platform, which you can use to automate some tasks. Our work that's going to be automated is called **workflow**, Tines is based on set of actions, we're going to use HTTP request, Trigger and Event Transformation actions.

Translating all of this into action, creating an HTTP request action that grabs the **search_id** from the searches tab as we've done before. 

![tines-1](https://i.imgur.com/jtIaqd2.png)

![sec-header](https://i.imgur.com/tWavpzU.png)

The URL contains the query and the path of the API call, The method is POST and we need to disable SSL verification as in my case my SIEM solution has a self-signed certificate, also in the headers section ***SEC*** is the authorization token we've created before so API calls can exist.

![response-1](https://i.imgur.com/He5pPxt.png)

In the response body, we can see the **search_id** of our query which then we can use later on to get the results.

![action-2](https://i.imgur.com/sUKk066.png)

In our second HTTP request action, we are going to retrieve the results of our query via the **search_id**, which in this case was passed before in the HTTP response body from the previous HTTP request action. It has the most of the attributes of the last HTTP request action but we're going to change the method to **GET** .

**search_id** path in the response body -> `qradar_query_body.search_id`

![response-2](https://i.imgur.com/cZ25lKV.png)

Here we can see that we've successfully retrieved all of the destination IP addresses from the previous query.


Okay, now we can check each IP address whether malicious or not, but there's a problem, we can't pass all of our response body parts at once we need to iterate over it as an array, so we're using the Event Transformation Action.

![action-3](https://i.imgur.com/XwxHCFv.png)

In the Event Transformation Action, we're selecting the Explode mode as we've said we need to iterate over a specific variable multiple of times in this case the variable is `destinationip` but to get to the destinationip we need to move through the response body in the path `qradar_search_results_body.flows`. 

With that we've successfully achieved our iteration process, now we pass each IP address to the VirusTotal API to check whether the IP address is malicious or not.

![action-4](https://i.imgur.com/MGYd5GH.png)

In this HTTP request action, we're communicating with the [VirusTotal API](https://docs.virustotal.com/reference/overview), in the URL we're specifying the IP address from our last Event Transformation Action, for each IP we're going to scan whether this IP is malicious or not.

![response-4](https://i.imgur.com/p5TUIlO.png)

In the response body, you can see that a vendor voted that this IP address is malicious, we can utilize this further and create a simple if condition whether a malicious IP address was detected we should fire an alert to set of email addresses.

![action-4](https://i.imgur.com/hslMesU.png)

In this case in case a malicious IP address was detected in the environment send an email, at a low-level, check whether in the VirusTotal response body in the total_votes part if a malicious was flagged with more or equal to 1 by a vendor, if so, send an email.

![mail-1](https://i.imgur.com/joRlMvJ.png)

Here we can see that our workflow was successful.

## Workflow Overview

![workflow](https://i.imgur.com/S7gDkiR.png)
