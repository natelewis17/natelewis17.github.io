---
layout: post
title:  "Wikiji: Data Collection"
author: Nate Lewis
description: How I collected data on kanji frequnecies for my project.
image: /assets/images/jp_writing.jpg
---

<h2>Introduction and Motivation</h2>

Japanese is widely considered to have one of the most complex writing systems with two unique syllabries hiragana   (ひらがな) and katakana (カタカナ) being used in conjunction with characters that originated in China known as kanji (漢字). While the process for learning hiragana and katakana is relatively straightforward for beginners with 46 characters in each syllabry, kanji is a little more complex. You see, there are approximately 50,000 kanji characters with over 2,000 being deemed necessary to being considered fluent. Making matters worse, kanji are considerably more complicated than the other writing systems with upwards of 20 strokes needed to create some of the more complex singular characters. As someone who has been studying actively for 2 years I can only claim to know about 1,000 kanji with confidence. The task of learning these 2,000 or more characters is daunting for any learner. So the question is: Where do I even begin?

The most obvious answer to this question is to start with the most common and therefore the most essential ones. But how does one know which are most common and is the even the best way to learn? My goal with this project is to help answer these questions by looking at the frequency of kanji usage on the website Wikipedia.com.

Data collection was done using the [MediaWiki API](https://www.mediawiki.org/wiki/API:Main_page) with special consideration given to the provided API Etiquette and Usage Guidelines. Additionally, I used the [Japanese Text Analysis API](https://rapidapi.com/rlemaigre/api/japanese-text-analysis). This API required an API key so I had to ensure not to include the key on any publically available page. For usage of both APIs I included a delay in my code to make sure that too many requests were not being made.

<h2>Collecting the Data</h2>

The first step was to create a function that using the WikiMedia API would select a random page from Japanese Wikipedia and then scrape specifically the kanji text of the page. This was done using the following code:

```python
def get_random_wikipedia_kanji():
    # Wikipedia API endpoint for a random page
    api_url = 'https://ja.wikipedia.org/w/api.php?action=query&list=random&rnlimit=1&format=json'

    response = requests.get(api_url)
    data = response.json()

    # Extract the title of the randomly selected page
    page_title = data['query']['random'][0]['title']

    # Construct the Wikipedia page URL using the title
    encoded_title = urllib.parse.quote(page_title)
    page_url = f'https://ja.wikipedia.org/wiki/{encoded_title}'

    print(f"Processing: {page_url}")

    response = requests.get(page_url)
    soup = BeautifulSoup(response.text, 'html.parser')

    # Extract kanji data from the Wikipedia page
    kanji_data = []
    for paragraph_tag in soup.find_all('p'):
        kanji_text = ''.join(char for char in paragraph_tag.text if char.isalnum())
        kanji_data.extend(kanji for kanji in kanji_text if '\u4e00' <= kanji <= '\u9fff')

    return kanji_data
```

After creating the function, I then needed to create a for loop so that I could get text data from multiple pages in quick succession. I decided that [343](https://www.youtube.com/watch?v=0jXTBAGv9ZQ) iterations would be sufficient for getting an idea of frequency. The code for the for loop is below:

```python
# Number of iterations
num_iterations = 343

# List to store kanji data from each iteration
all_kanji_data = []

# Perform the process 100 times
for _ in range(num_iterations):
    kanji_data = get_random_wikipedia_kanji()
    all_kanji_data.extend(kanji_data)
    # Add a delay to avoid making too many requests in a short time
    time.sleep(1)
```

I then created a DataFrame using the following code:

```python
# Create a dictionary to store the frequency of each kanji character
kanji_frequency_dict = {}

# Count the frequency of each kanji character
for kanji in all_kanji_data:
    kanji_frequency_dict[kanji] = kanji_frequency_dict.get(kanji, 0) + 1

# Create a dataset with columns for kanji and frequency
kanji_frequency_df = pd.DataFrame(list(kanji_frequency_dict.items()), columns=['Kanji', 'Frequency'])

# Sort Kanji by Frequency (Descending)
kanji_frequency_df = kanji_frequency_df.sort_values(by='Frequency', ascending=False)
kanji_frequency_df = kanji_frequency_df.reset_index(drop=True)
```

<h2>To Be or Not to Be (Joyo)</h2>

As previously mentioned a person wanting to achieve "fluency" in Japanese needs to learn over 2,000 kanji. This number of 2,000 (or more specifically 2,136) comes from the joyo kanji guide created by the Japanese Ministry of Education. I decided that as interesting for a learner as what are the most common kanji is the question of which of the "essential" kanji don't appear at all. Thus I decided to farther add onto my DataFrame all of the kanji from the joyo list that didn't appear at all on the 343 Wikipedia pages, and to do so I again went to Wikipedia and their [list of joyo kanji](https://en.wikipedia.org/wiki/List_of_j%C5%8Dy%C5%8D_kanji). The following code was used to create a new DataFrame and then merge the two DataFrames together:

```python
# Wikipedia URL for the Joyo kanji list
url = "https://en.wikipedia.org/wiki/List_of_j%C5%8Dy%C5%8D_kanji"

# Send an HTTP request to the URL
response = requests.get(url)

# Check if the request was successful (status code 200)
if response.status_code == 200:
    # Parse the HTML content of the page
    soup = BeautifulSoup(response.content, "html.parser")

    # Find the table containing Joyo kanji
    joyo_table = soup.find("table", {"class": "wikitable"})

    # Convert the HTML table to a DataFrame using pandas
    joyo_df = pd.read_html(str(joyo_table))[0]

    joyo_df.rename(columns={"New (Shinjitai)": "Kanji"}, inplace=True)

    # Display the DataFrame
    print(joyo_df)
else:
    print(f"Failed to retrieve content. Status code: {response.status_code}")

kanji_set_frequency = set(kanji_frequency_df["Kanji"])
kanji_set_joyo = set(joyo_df["Kanji"])

# Find the set difference (kanji in joyo_df but not in kanji_frequency_df)
kanji_not_in_frequency = kanji_set_joyo - kanji_set_frequency

# Convert the result to a list
kanji_not_in_frequency_list = list(kanji_not_in_frequency)

missing_kanji_df = pd.DataFrame({"Kanji": kanji_not_in_frequency_list, "Frequency": [0] * len(kanji_not_in_frequency_list)})

# Concatenate the new DataFrame to kanji_frequency_df
kanji_frequency_df = pd.concat([kanji_frequency_df, missing_kanji_df], ignore_index=True)
```

I also decided to add a column called "In_Joyo." This column simply reports whether a kanji is in the joyo list or not. I did this to answer the question of which non-joyo kanji are appearing the most frequently and therefor should be prioritized by learners.

<h2>What Do They Mean?</h2>

Okay so now I have a big long list of a couple thousand chinese characters, but how are we supposed to get anything from this list if we don't even have data on basic info about the kanji like their stroke counts or even what they mean in English. This is when the Japanese Text Analysis API became important. Using this API I was able to create new columns for my DataFrame that would be essential to making the data meaningful for a nonfluent learner and for answering my questions when it came time for EDA. Using this API I was able to create new columns pertaining to:
* English Meanings of the Concepts the Kanji Represent
* Stroke Count
* Grade at Which Kanji is Taught to Japanese Students
* JLPT Level (JLPT is a series of 5 exams for foreigners to certify competency in Japanese)
* Radicals (Radicals are the building blocks of kanji. For example 時、待、持 all share the 寺 radical).
* Newspaper Ranking
* Novel Ranking (the importance of the newspaper and novel ranking will be explored more deeply in the EDA blog post)
Due to the limit on this API of 1000 queries per hour I had to create 3 different DataFrames containing this new data which I then merged together. The code for creating the first of these three can be seen below:

```python
url = "https://japanese-text-analysis.p.rapidapi.com/"

with open("kanji_apikey.txt", "r") as key_file:
    api_key = key_file.read().strip()

headers = {
    "x-rapidapi-key": api_key,
    "x-rapidapi-host": "japanese-text-analysis.p.rapidapi.com",
    "Content-Type": "application/json"
}

kanji_list = kanji_frequency_df_subset1["Kanji"].tolist()

# Initialize lists to store data
literal_list = []
frequency_list = []
grade_list = []
jlpt_list = []
meanings_list = []
radical_list = []
ranking_list = []
stroke_count_list = []

# Iterate through kanji characters
for kanji in kanji_list:
    # Update the GraphQL query for the specific kanji
    payload = {
        "query": f"""
            query MyQuery {{
                kanji {{
                    literal
                    frequency
                    grade
                    jlpt
                    meanings
                    radical
                    ranking
                    stroke_count
                }}
            }}
        """
    }

    # Make the request
    response = requests.post(url, json=payload, headers=headers)

    # Handle the response
    if response.status_code == 200:
        data = response.json().get("data", {}).get("kanji", {})
        # Append data to lists
        literal_list.append(data.get("literal", ""))
        frequency_list.append(data.get("frequency", ""))
        grade_list.append(data.get("grade", ""))
        jlpt_list.append(data.get("jlpt", ""))
        meanings_list.append(data.get("meanings", []))
        radical_list.append(data.get("radical", ""))
        ranking_list.append(data.get("ranking", ""))
        stroke_count_list.append(data.get("stroke_count", ""))
    else:
        print(f"Error for {kanji}: {response.status_code}, {response.text}")

    # Introduce a 1-second delay
    time.sleep(1)

# Create a DataFrame
top_df = pd.DataFrame({
    "Kanji": literal_list,
    "Frequency": frequency_list,
    "Grade": grade_list,
    "JLPT": jlpt_list,
    "Meanings": meanings_list,
    "Radical": radical_list,
    "Ranking": ranking_list,
    "Stroke_Count": stroke_count_list,
})
```

I then concatenate these three DataFrames and the immediately merged the result with my existing DataFrame containing the data I got from Wikipedia like this:

```python
# Concatenate the DataFrames along the rows
kanji_data_df = pd.concat([top_df, middle_df, bottom_df], ignore_index=True)

# merging my two DataFrames
kanji_df = pd.merge(kanji_frequency_df, kanji_data_df, on="Kanji", how="left", suffixes=('_freq', '_data'))
```

<h2>Conclusion</h2>

The final result after all of that work is a DataFrame with 2519 rows and 11 columns. The code provided above is a simplified version of my code used to create my DataFrame so if you would like to see all of the code used or the .csv file I made you can find that in my GitHub repository [here](https://github.com/natelewis17/STAT386_Project).

Thank you for reading through my blog post. Even if you are not interested in learning Japanese at all, I hope you got something of this post. I'd like to finish by encouraging all future statisticians to work with data that pertains to their interests and also a reminder to always be ethical when web scraping.
さよなら兄弟!
