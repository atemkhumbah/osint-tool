# osint-tool
OSINT investigation tool designed to simplify the process of gathering and analyzing publicly available information
import requests
from bs4 import BeautifulSoup
import sqlite3
import os
#I am particularly focused on implementing web scraping and data aggregation functionalities. My initial challenge will be to ensure effective data collection from various online sources. I plan to create a proof of concept for web scraping using Beautiful Soup and Scrapy. 

# Function to set up the database connection
def setup_database():
    # Delete the existing database file if it exists
    if os.path.exists('books.db'):
        os.remove('books.db')
    
    conn = sqlite3.connect('books.db')
    c = conn.cursor()
    
    # Create a table for books
    c.execute('''
    CREATE TABLE IF NOT EXISTS books (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT,
        price TEXT,
        author TEXT,
        availability TEXT
    )
    ''')
    
    return conn, c

# Fetch the HTML content of a page
def fetch_page(url):
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }
    try:
        response = requests.get(url, headers=headers)
        response.raise_for_status()  # Raise an error for bad responses
        print(f"Successfully fetched {url}")
        return response.text
    except requests.RequestException as e:
        print(f"Error fetching {url}: {e}")
        return None

# Parse the HTML and extract book data
def parse_books(html):
    soup = BeautifulSoup(html, 'html.parser')
    return soup.find_all('article', class_='product_pod')

# Save book data to the database
def save_books_to_db(cursor, books):
    for book in books:
        title = book.h3.a.get('title', 'No title found')
        price_element = book.find('p', class_='price_color')
        price = price_element.text if price_element else 'No price found'
        
        # New data: author and availability
        author = 'No author found'  # Placeholder for author
        availability = book.find('p', class_='instock availability').text.strip() if book.find('p', class_='instock availability') else 'No availability found'

        # Print extracted data
        print(f'Adding book: Title: "{title}", Price: {price}, Author: {author}, Availability: {availability}')
        
        # Insert data into the database
        cursor.execute('INSERT INTO books (title, price, author, availability) VALUES (?, ?, ?, ?)', (title, price, author, availability))

def main():
    # Set up the database
    conn, cursor = setup_database()

    # Base URL for scraping
    base_url = 'http://books.toscrape.com/catalogue/page-{}.html'
    page_number = 1

    while True:
        url = base_url.format(page_number)
        html_content = fetch_page(url)

        if not html_content:
            print("No more pages to scrape or failed to retrieve the page.")
            break

        books = parse_books(html_content)
        if not books:
            print("No books found on this page.")
            break

        # Save the extracted books to the database
        save_books_to_db(cursor, books)
        
        page_number += 1

    # Commit changes and close the database connection
    conn.commit()
    conn.close()
    print("Data saved to books.db")

# Execute the main function
if __name__ == "__main__":
    main()
#Saves as book_scraper.py
Used metplolib to visualize data from this book website, here is my code
import sqlite3
import matplotlib.pyplot as plt

# Connect to the database
conn = sqlite3.connect('books.db')
cursor = conn.cursor()

# Query to get the price of each book
cursor.execute("SELECT price FROM books")
prices = cursor.fetchall()

# Convert prices from string to float (removing the currency symbol)
price_values = []
for price in prices:
    price_value = float(price[0][1:].replace('£', '').strip())  # Skip the currency symbol (e.g., £)
    price_values.append(price_value)

# Create price bins for histogram
bins = [0, 10, 20, 30, 40, 50, 60]  # Adjust based on your price range
plt.hist(price_values, bins=bins, edgecolor='black')

# Add titles and labels
plt.title('Distribution of Book Prices')
plt.xlabel('Price (£)')
plt.ylabel('Number of Books')

# Show the plot
plt.show()

# Close the database connection
conn.close()
#Saved as visualize_books.py
