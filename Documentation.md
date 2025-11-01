# AI Automation Assignment: Multi-Agent Content Pipeline

This document outlines the design, implementation, and logic of a multi-agent content pipeline built in n8n. [cite_start]The workflow automates the entire process from topic research to content generation (blog and video) and submission for review, as per the assignment requirements[cite: 22, 73].

## Agent 1: Content Research Agent

* [cite_start]**Goal:** Discover trending topics related to "AI Automation" from Google Trends and YouTube[cite: 25].
* **Nodes Used:** HTTP Request (for Google Trends), YouTube, Merge, Remove Duplicates.

### Node Configuration:

1.  [cite_start]**HTTP Request (Google Trends):** [cite: 27, 31]
    * **Purpose:** Fetches daily trending topics from Google Trends, as a direct node was not available.
    * **Configuration:** Uses a non-API endpoint (e.g., `https://trends.google.com/trends/api/dailytrends`) with parameters for the "AI Automation" niche.
    * [cite_start]**Error Handling:** A 'Retry On Fail' setting is enabled[cite: 75].

2.  **Clean Trends (Code):**
    * **Purpose:** Parses the raw JSON output from the Google Trends HTTP request. It extracts and formats the relevant topic titles into a clean list.

3.  [cite_start]**YouTube Node:** [cite: 28, 32]
    * **Purpose:** Searches YouTube for top trending videos related to "AI Automation".
    * **Configuration:** Uses the YouTube Data API v3 credential. The 'Search: list' operation is used to find 10-15 recent, popular videos.
    * [cite_start]**Error Handling:** 'Retry On Fail' is enabled[cite: 75].

4.  **Clean YouTube (Code):**
    * **Purpose:** Parses the output from the YouTube node. It maps over the `items` array and extracts the `video.snippet.title` for each video, creating a clean list of topics.

5.  [cite_start]**Merge Node:** [cite: 33]
    * **Purpose:** Combines the topic lists from both Google Trends and YouTube into a single dataset.
    * **Mode:** `Append`.

6.  [cite_start]**Remove Duplicates Node:** [cite: 33]
    * **Purpose:** De-duplicates the merged list to ensure each topic is unique before passing it to the next agent.

## Agent 2: Prompt Agent

* [cite_start]**Goal:** Generate creative blog and video prompts for each trending topic[cite: 37].
* **Nodes Used:** Code (JavaScript).

### Node Configuration:

1.  [cite_start]**Clean\_Prompts (Code):** [cite: 43]
    * **Purpose:** This node iterates through each topic received from Agent 1. For each topic, it programmatically generates two distinct prompts:
        1.  [cite_start]`blog_prompt`: A detailed prompt for generating a comprehensive blog post[cite: 48].
        2.  [cite_start]`video_prompt`: A prompt designed to create a short, engaging script for a video[cite: 49, 55].
    * **Logic:** The prompts are stored in an object along with the original topic, creating a (topic + prompt) pair for the next agent.

## Agent 3: Content Creator Agent

* [cite_start]**Goal:** Generate a full blog post and a video for each topic + prompt pair[cite: 46].
* **Implementation:** This agent is split into two loops to handle API rate limiting and server overload errors (like `503 Service Unavailable`).

### Part 1: Blog Post Generation

1.  **Loop Over Items (Blog Loop):**
    * **Purpose:** Processes each (topic + prompt) pair one by one to prevent API overload.
    * **Configuration:** `Batch Size` is set to `1`.

2.  [cite_start]**Google Gemini (Generate Blog Post):** [cite: 48]
    * **Purpose:** Generates the blog post text.
    * **Configuration:**
        * **Resource:** `Text`, Operation: `Message a Model`.
        * **Prompt:** Uses an expression `{{ JSON.stringify($json.blog_prompt) }}` to correctly pass the prompt object as a string, preventing a `400 Bad Request` error.
        * **Settings:** `Retry On Fail` is set to **ON** with 3 retries and a 5000ms (5-second) wait.

3.  **Edit Fields1:**
    * **Purpose:** Merges the original input data (`topic`, `video_prompt`, `blog_prompt`) from the loop with the new output (`content`) from the Gemini node. This is critical because the Gemini node drops the input data.
    * **Configuration:** `Include Other Input Fields` is **ON**. Fields for `topic`, `video_prompt`, and `blog_prompt` are manually mapped using expressions like `{{ $('Loop Over Items').item.json.topic }}`.

4.  **Wait1:**
    * **Purpose:** Adds a 2-second delay after each blog post generation to further prevent API rate limiting.

5.  **Code in JavaScript (Blog Clean-up):**
    * **Trigger:** Connects to the `done` output of the blog loop.
    * **Mode:** `Run Once for All Items`.
    * **Purpose:** Cleans the final array of 37 items. It extracts the text from the nested `content.parts[0].text` object and places it into a new, clean `blog_content` field, while preserving the `topic` and `video_prompt`.

### Part 2: Video Generation

1.  **Loop Over Items (Video Loop):**
    * **Purpose:** Takes the cleaned output from the blog generation (now containing `topic`, `video_prompt`, `blog_content`) and loops through it one by one.
    * **Configuration:** `Batch Size` is set to `1`.

2.  [cite_start]**Google Gemini (Generate Video):** [cite: 49, 55]
    * **Purpose:** Generates a video based on the `video_prompt`. (Simulating the Google VEO 3 API as per assignment).
    * **Configuration:**
        * **Resource:** `Video`, Operation: `Generate a video`.
        * **Prompt:** Uses an expression to pass the `video_prompt`: `{{ $json.video_prompt }}`.
        * [cite_start]**Settings:** `Retry On Fail` is **ON**[cite: 75].

3.  **Edit Fields (Video Loop):**
    * **Purpose:** Merges the original data (`topic`, `video_prompt`, `blog_content`) with the new video generation output (e.g., `file_id`, `url`).
    * **Configuration:** `Include Other Input Fields` is **ON**. Fields are re-mapped from the video loop's input item, just like in the blog loop.

4.  **Wait (Video Loop):**
    * **Purpose:** Adds a 2-second delay to prevent API rate limiting for video generation.

## Agent 4: Content Submission Agent

* [cite_start]**Goal:** Submit the final collated content to a Google Sheet for human review[cite: 60].
* **Nodes Used:** Google Sheets.

### Node Configuration:

1.  [cite_start]**Google Sheets:** [cite: 62]
    * **Trigger:** Connected to the `done` output of the **Video Generation Loop**.
    * **Operation:** `Append row in sheet`.
    * **Configuration:**
        * **Spreadsheet ID:** The ID of the target Google Sheet was pasted.
        * **Sheet Name:** `Sheet1` (or the relevant sheet name) was selected.
        * [cite_start]**Column Mapping:** Fields were mapped manually as required by the assignment [cite: 66-71]:
            * `Topic` -> `{{ $json.topic }}`
            * `Prompt` -> `{{ $json.video_prompt }}`
            * `Blog Content` -> `{{ $json.blog_content }}`
            * `Video Link` -> `{{ $json.video_data_field }}` (This expression points to the field containing the video link from the Gemini video node, e.g., `$json.url` or `$json.file_id`).
            * `Timestamp` -> `{{ $now.toFormat('dd/MM/yyyy HH:mm') }}`
            * `Status` -> (Static Value) `Pending Review`

## API Setup Summary

1.  **Google Trends (HTTP Request):** No API key required. Used a public-facing URL.
2.  **YouTube Data API:** A project was created in Google Cloud Platform. The YouTube Data API v3 was enabled, and an API key was generated and used in the YouTube node.
3.  **Google Gemini API:** An API key was generated from Google AI Studio and used in the Google Gemini node credentials.
4.  **Google Sheets API:** OAuth 2.0 credentials were created in Google Cloud Platform to allow n8n to authenticate and append data to the sheet.