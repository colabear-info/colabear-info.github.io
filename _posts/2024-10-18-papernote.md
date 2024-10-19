---
title: "Llava with his friends"
last_modified_at: 2024-10-18T11:51:33-05:00
categories:
  - Note
tags:
  - LLM
---

Paper read:
The predicted token could be anything, 
from the motion of the robot (OpenVLA: An Open-Source Vision-Language-Action Model) to the semantic meaning of a picture (text4seg). Also, to avoid generate text repeatitely, they compress the similar token into the pattern like token, token_number. Here, they use a property of LLM that could generate structured output. 

My understanding is that the idea between matching between the vision encoder and llm decoder is very similiar to the idea of unsupervised machine learning translation method. 

Also go throught a method that published by Zhe Gan, that incoporate the location information in the CLIP pretraining, which could be used for fine-grained matching between the image and texts. Why don't I come up with this idea? 

A easy combination is that, what if we use the clip trained from Zhe with location information to do such a image caption problem to see if it performs better? How does it compare with the location embedding that we added to the image. That's an open question to discuss (which also requires a lot of computation resources).

One question I asked myself today, do I really like doing research? Do I just don't like the feeling that I am left behind? Everyone has their own pace, and I should have my own goal. BTW, I got rejected by Wayve today. Feeling bad but know the reason. If I really want to do research, I should start reading paper. I should keep in mind, only one idea might word out of one hundred, do not be afraid of failure.

I do hope that someone could tell me that, when they provide me with that opportunity, it is them that should take the responsibility of a failure candidate. 

Glad that I write things down here, second day of daily update. 



