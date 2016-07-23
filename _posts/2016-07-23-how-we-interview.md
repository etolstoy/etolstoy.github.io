---
layout: post
title: How We Interview
description: How we interview - a well defined set of technical questions, trusted system to collect results and interviewers rotation.
---

We have a team over 25 iOS developers at the moment. Despite this is a pretty large number we're still looking for more people to come. In short, that means three things:

- We've got a lot of interviews every week
- Many collegues want to take part in this process and sharpen their interviewing skills
- You can apply too - our team and projects are really cool!

Till the last time this process was pretty chaotic. We didn't clearly understand what to ask, how to process the results and how to compare different candidates for one role. So I spent some time on researching and organizing it with a great help from my collegues - and we developed a noticable system.

<!--more-->

So, let's dive into our problems one by one.

#### What to ask

Literally every developer has his own understanding of what it is - being a good software engineer. Everyone asks his set of questions polished by years of work. Some like to talk hours about data structures and algorithms, others - about UIKit and other system framework classes. Sometimes there wasn't enough time for technical questions - the candidate had too much to tell about his previous work experience. Some time ago we even had a collegue who completely refused to hire anyone who has worked in a bank.

That resulted in a very serious problem. Whether we hire a candidate or not depended completely on who interviewed him and what questions he was asked. We didn't have an objective way to measure someone's skills.

In June me and my collegue had talks at Mobius conference. The road from Moscow to St Petersburg is pretty long, so we had quite enough time to determine what parts of knowledge we want to query during the interview. We got the following list:

- Data structures and algorithms
- Parallel programming
- Patterns and principles
- Architecture of mobile applications
- Testing
- Memory management
- Data persistance
- UIKit

The interview is limited in time - we don't want to exhaust a candidate. So each section has not that many questions - typically from three to six. Some of them require just a short answer, others - the ability to address his previous work experience. Besides questions we prepared a couple of more complex quizes - from typical algorithms tasks to refactoring massive view controller on the go. Of course, at first there was to much questions, so we continuously reduced the list to make it possible to finish the interview in two hours.

We hire developers of a different qualification - from junior to senior. The set of questions varies for all these grades. Usually at the beginning of the interview we already know who we want to hire. If not, we ask some sighting questions ar first - and after it choose the appropriate grade.

#### Processing results

A well defined set of questions helped us to be confident that we aquire rough data about each candidate's understanding of all topics interesting for us. However, we still were not able to collect this data and compare different candidates. What's even worse - it was very hard to abstract from personal impression of the candidate and remember his level of CoreData knowledge even on the next day. When there are ten interviews a week it's impossible to remember such details.

So, the next step was to think of a way to measure the results. We decided to use a five-point scale for each answer with specific coefficients for different grades. As you can see in the sidebar, I'm in task automation - so we transferred all the questions to a Google Form and added a rating scale to them. With this approach the interviewers don't have to remember the exact answers of a candidate after the interview - they fill the form continuously during the meeting.

![Interviewing form](/public/img/posts/interviewing-form.png)

Google Form is a great instrument of collecting data, but it requires some more efforts to properly interpretate and represent it. I dived into an ActionScript and wrote a thing that merges the marks of all the interviewers into a single value, sums up all the answers for a section and prints it into a table.

![Interviewing table](/public/img/posts/interviewing-table.png)

Finally, we got a more objective way of scoring candidates. This approach got some pleasant side-effects. We are able to see how a candidate improved his knowledge if he comes to us again after a year or so. That's super handy because it can tell if a person is interested in learning and improving himself. Besides it, candidates can receive a more representative feedback from us - what areas they are good at, and what topics they should learn.

#### Selecting interviewers

Our team is huge and at least half of it wants to participate in interviews. To be fair and let everyone take part we're going to create another Google Form - this time with interviewers rotation.

It would be very simple - you just select who and when took part in the interview. This data is represented as a sorted spreadsheet, where the highest person in the list is the one who you should take to the meeting.

Of course, not everyone can act as an interviewer. Our constraints are: you should be of the middle grade at least and you should participate in our training. During this training two intern interviewers ask each other questions from our list, while their more experienced collegue helps them.

#### Summary

Our interviewing system consists of three core ideas:
- A well-defined set of questions differentiated for all programming grades,
- A trusted system that collects results and represents them in a convinient and expressive way,
- Continuous rotation of interviewers.

We moved to this system not very long ago, but it already gives us nice results. In a year or so after collecting more data I'll write another post with some statistics and detailed conclusions.