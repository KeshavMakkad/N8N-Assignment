# Problem Statement

## Background

I work at Reginald Men, a men's grooming brand under Honasa Consumer. At some point I was asked to go through all the negative reviews on our Shopify store and figure out what customers were actually complaining about — which products, what issues, what we should fix.

## What I Was Doing Before

The process I had at the time looked like this:

1. Download the reviews from Shopify as a CSV
2. Filter out the negative ones manually in Google Sheets
3. Paste them into Claude with a prompt
4. Read the output and pass it along to the team

Claude did the job fine. The problem was everything around it — I had to remember to do it, write the prompt from scratch every time, and there was no consistent output format or anywhere the results were actually stored.

## Why This Was a Problem

- I had to manually trigger it every single time
- The output changed depending on how I wrote the prompt
- Nothing was stored — insights lived in a chat window
- There was no approval step before it reached the team
- It didn't scale as review volume grew

## Goal

Automate the whole thing end to end — pull the data, filter it, classify each review, generate insights on what's going wrong and why, get a human to sign off, and write the final report to a Google Doc automatically.

## Expected Output

A timestamped Google Doc with a breakdown of complaints by category, AI-generated analysis of what's going wrong and why, and a full list of reviews with their classifications.