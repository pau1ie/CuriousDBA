---
title: "Oracle Sales: Cloud Analytics, Blockchain, IoT, AI - HEUG EMEA 2019"
date: 2019-11-14T14:33:50Z
tags: ['HEUG2019', 'Oracle', 'Sales','Cloud']
---

Oracle puts effort into selling at HEUG, which makes sense because we are all users of Oracle software. It is interesting
to know what is out there, but the likelihood is we will never get to use most of it. Also, Oracle pushes cloud which I
find quite depressing because I don't know where I fit in to the new cloud paradigm.

I made notes from a couple of these sessions, which are quite short, so I will combine them here.

## Oracle Cloud Analytics
In my mind this is like OBIEE, but moved on a bit to be more user friendly, more drag and drop. It looks like
Google Analytics, and is supposed to be just as friendly. The sales man promised a lot, that you could upload
a spreadsheet, then it would automatically identify all the columns, and tell you about them, and tell you
interesting information.

The data visualisation is very pretty, and it does seem user friendly. However, it strikes me that you still need
to understand statistics, the underlying dataset(s) and the limitations of each to understand what it is 
telling you. The dataset used to demonstrate was a list of 
employees and contained a column called attrition. This doesn't make sense to me because an employee is either
working or has left. Attrition is something that pertains to a group over time. He demonstrated attrition against
pay, which again doesn't make sense to me because if an employee has left, they are no longer getting pay rises.
My colleague pointed out that while the tool can find correlation between whether people have left and other
data points, this doesn't mean that the correlation found is actually the reason for it.

He demonstrated importing a spreadsheet and the tool detecting what the data in the columns is, and describing information about
them. My suspicion is that this will miss  the subtleties of the information in the spreadsheet, and end up being misleading.

It seems it can build reports from natural language queries, and also give natural language insights into data.

So while it is a pretty tool, and seems to reduce  the technical knowledge required, it will still require statistics
and knowledge of the underlying data to be able to use effectively.


## Blockchain IoT and AI

The last session of the conference was stuffed with  buzzwords to keep us all interested. 
It was led by Peter Szegedi - senior technology solution engineer - Oracle Digital EMEA. 

Peter explained the life cycle of these products. They start off as a platform, I assume where they deliver libraries 
for to work with things, to applications where presumably it is ready to use in specific cases. It them moves to 
specific applications when it is mature. He didn't explain what happens to customers who are using the platform services
and don't want to use the applications.

### AI (Artificial Intelligence)

He considers AI is a moving target. Previously a chess playing program was AI, but this isn't the case any more. 
Now we talk about machine and deep learning as AI.
He Listed a load of algorithms that Oracle use. For some it is not a stand alone service but works across a number of applications. 
They all have cloud in their name, Oracle marketing is consistent on this at least! Though I noticed a number
in the database. Some examples were [Spark](https://spark.apache.org/), [R](https://www.r-project.org/), Data Miner, some SQL functions.

### Blockchain

He mentioned storing exam results as a use case for blockchain. This is something that was mentioned earlier in a keynote. He considered that
this was an ideal use of blockchain. Later an insightful question mentioned that storing this information in one
suppliers cloud controlled by one institution effectively undoes the advantage of blockchain - the fact that "everyone" agrees
on the history. If Oracle Cloud controls most of the copies, it can rewrite history. Peter acknowledged this and said the
blockchain would need to be spread across several institutions, but didn't address the point that Oracle are selling this as a
cloud solution. Another questioner asked about the environmental impact of blockchain, as it uses enormous amounts of energy, which
again was acknowledged as an issue but not addressed.

Blockchain is a platform service which suggests Oracle haven't quite worked out the best way use it yet.

### IOT (Internet of Things)

Peter gave an example from manufacturing. Knowing which parts were installed in which cars allows a company to 
know which cars to recall due to a faulty part.

Another example is a haulage company tracking ships knowing when they will dock and having transport capacity to sell at the right time so they made loads of money.

To get full value of IoT, devices have to talk to each other and exchange information.

Passive IoT is e.g. a bar code, active IoT always connected, e.g. self driving cars communicating their intentions with each other. 
Students might not want to be tracked in this way. Maybe they could be encouraged to share the information?

IoT is no longer a platform service - it is in the applications.

### Conclusion

At the end was the obligatory advert for Oracle Cloud. There is a 60 day trial. Some services are free after this period. 
You can play with the blockchain in the 60 day trial. This was also mentioned in the cloud analytics session. Apparently
most people that sign up for the trial don't make the time to play with the technologies before the period expires, which
is understandable I suppose.