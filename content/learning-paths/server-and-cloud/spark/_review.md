---
# ================================================================================
#       Edit
# ================================================================================

# Always 3 questions. Should try to test the reader's knowledge, and reinforce the key points you want them to remember.
    # question:         A one sentence question
    # answers:          The correct answers (from 2-4 answer options only). Should be surrounded by quotes.
    # correct_answer:   An integer indicating what answer is correct (index starts from 0)
    # explanation:      A short (1-3 sentence) explanation of why the correct answer is correct. Can add aditional context if desired


review:
    - questions:
        question: >
            Spark can only run on Hadoop staorage?
        answers:
            - "True"
            - "False"
        correct_answer: 2                     
        explanation: >
            Apache Spark can run without Hadoop, standalone, or in the cloud.
            
    - questions:
        question: >
           Cloud Foundry is the major competitors of Terraform?
        answers:
            - "True"
            - "False"
        correct_answer: 2                    
        explanation: >
            Ansible is the major competitors of Terraform.
               
# ================================================================================
#       FIXED, DO NOT MODIFY
# ================================================================================
title: "Review"                 # Always the same title
weight: 20                      # Set to always be larger than the content in this path
layout: "learningpathall"       # All files under learning paths have this same wrapper
---



