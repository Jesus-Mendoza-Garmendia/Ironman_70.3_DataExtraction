# Ironman_70.3_DataExtraction

This repository contains Python code designed to extract historical data from Endurance Data for the Ironman 70.3 Augusta race. By scraping multiple years of data from various URLs, this project provides a comprehensive dataset that can be used for in-depth analysis and performance prediction. The extracted data is organized into a CSV file, ready for further processing and analysis in tools like R or Python.

``` Python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import logging

# Function to scrape data for multiple years and create one CSV
def fetch_race_results(years_data):
    # Set up logging
    logging.basicConfig(level=logging.INFO)
    
    # List to hold all competitor data
    all_competitors = []
    
    # Loop through each year and its corresponding details
    for year, details in years_data.items():
        url = details['url']
        total_pages = details['pages']
        george_stewart_adjustment = details.get('george_stewart_adjustment', False)

        for page in range(1, total_pages + 1):
            full_url = f"{url}{page}/"
            logging.info(f"Scraping {year} - page {page} from {full_url}")
            
            # Fetch and parse the page content
            response = requests.get(full_url)
            soup = BeautifulSoup(response.content, 'html.parser')

            # Find table rows containing competitor data
            rows = soup.find_all('tr')

            for row in rows[1:]:  # Skip the header row
                columns = row.find_all('td')
                if len(columns) > 0:
                    try:
                        division = columns[5].text.strip() if columns[5].text.strip() else "N/A"
                        swim_time = columns[7].text.strip() if columns[7].text.strip() else "N/A"
                        bike_time = columns[8].text.strip() if columns[8].text.strip() else "N/A"
                        run_time = columns[9].text.strip() if columns[9].text.strip() else "N/A"

                        # Adjust George Stewart's bike time for 2023
                        if george_stewart_adjustment and 'George Stewart' in columns[3].text.strip() and bike_time == "00:00:00":
                            bike_time = "02:52:11"  # Correct bike time for 2023

                        competitor_data = {
                            'Year_of_Race': year,
                            'Name': columns[3].text.strip() if columns[3].text.strip() else "N/A",
                            'Bib': columns[4].text.strip() if columns[4].text.strip() else "N/A",
                            'Country': columns[6].text.strip() if columns[6].text.strip() else "N/A",
                            'Division': division,
                            'Gender': 'Team' if "MFRELAY" in division else 'Male' if 'M' in division else 'Female' if 'F' in division else "N/A",
                            'Gender_Rank': columns[1].text.strip() if columns[1].text.strip() else "N/A",
                            'Division_Rank': columns[2].text.strip() if columns[2].text.strip() else "N/A",
                            'Overall_Time': columns[10].text.strip() if columns[10].text.strip() else "N/A",
                            'Overall_Rank': columns[0].text.strip() if columns[0].text.strip() else "N/A",
                            'Swim_Time': swim_time,
                            'Bike_Time': bike_time,
                            'Run_Time': run_time,
                            'Finish_Status': 'Finisher'
                        }
                        all_competitors.append(competitor_data)
                    except IndexError:
                        logging.error(f"Error in row: {row}")
        
        # Print year confirmation and row count after each year's data collection
        print(f"Year: {year} - Total rows collected so far: {len(all_competitors)}")
    
    # Create a DataFrame from all competitors
    df = pd.DataFrame(all_competitors)
    
    # Confirm data for each year and column types
    print(df.head(20))  # Show a sample of the first 20 rows
    print(df['Year_of_Race'].unique())  # Check all years in DataFrame
    print(df.dtypes)  # Confirm data types

    # Convert time columns to timedelta for ranking calculations
    df['Swim_Time'] = pd.to_timedelta(df['Swim_Time'])
    df['Bike_Time'] = pd.to_timedelta(df['Bike_Time'])
    df['Run_Time'] = pd.to_timedelta(df['Run_Time'])

    # Apply ranking calculations within each year
    df['Swim_Rank'] = df.groupby('Year_of_Race')['Swim_Time'].rank(method='min', ascending=True).astype(int)
    df['Bike_Rank'] = df.groupby('Year_of_Race')['Bike_Time'].rank(method='min', ascending=True).astype(int)
    df['Run_Rank'] = df.groupby('Year_of_Race')['Run_Time'].rank(method='min', ascending=True).astype(int)

    # Format time columns to show HH:MM:SS without days for consistency
    df['Swim_Time'] = df['Swim_Time'].dt.components.apply(lambda x: f"{int(x['hours']):02}:{int(x['minutes']):02}:{int(x['seconds']):02}", axis=1)
    df['Bike_Time'] = df['Bike_Time'].dt.components.apply(lambda x: f"{int(x['hours']):02}:{int(x['minutes']):02}:{int(x['seconds']):02}", axis=1)
    df['Run_Time'] = df['Run_Time'].dt.components.apply(lambda x: f"{int(x['hours']):02}:{int(x['minutes']):02}:{int(x['seconds']):02}", axis=1)

    # Save to one consolidated CSV (uncomment below when ready to save)
    df.to_csv('Ironman_703_augusta_combined_results.csv', index=False)
    logging.info("Combined data saved to Ironman_703_augusta_combined_results.csv")

# Dictionary to hold URLs and page counts for each year
race_years = {
    2023: {'url': 'https://www.endurance-data.com/en/results/928-ironman-703-augusta/all/', 'pages': 36, 'george_stewart_adjustment': True},
    2022: {'url': 'https://www.endurance-data.com/en/results/750-ironman-703-augusta/all/', 'pages': 38},
    2021: {'url': 'https://www.endurance-data.com/en/results/589-ironman-703-augusta/all/', 'pages': 46},
    2019: {'url': 'https://www.endurance-data.com/en/results/448-ironman-703-augusta/all/', 'pages': 53},
    2018: {'url': 'https://www.endurance-data.com/en/results/290-ironman-703-augusta/all/', 'pages': 55},
    2017: {'url': 'https://www.endurance-data.com/en/results/84-ironman-703-augusta/all/', 'pages': 49}
}

# Call the function with all years' data
fetch_race_results(race_years)
```
![image](https://github.com/user-attachments/assets/8075ab43-e767-4ec3-8ec4-ec64b3e9e16d)
