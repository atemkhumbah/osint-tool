# Project one: NEDA : osint-tool
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
import tkinter as tk
from tkinter import messagebox, ttk
from tkinter import PhotoImage
import sqlite3
import requests
import time

# Function to get publication date using Open Library API
def get_publication_date(title):
    url = f'https://openlibrary.org/search.json?title={title}'
    response = requests.get(url)
    data = response.json()

    if 'docs' in data and data['docs']:
        first_result = data['docs'][0]
        publication_date = first_result.get('first_publish_year', 'Unknown')
        return publication_date
    return 'Unknown'

# Function to update the database with the publication date
def update_publication_dates():
    conn = sqlite3.connect('books.db')
    cursor = conn.cursor()

    cursor.execute("SELECT id, title FROM books WHERE publication_date IS NULL")
    books = cursor.fetchall()

    for book in books:
        book_id = book[0]
        title = book[1]

        publication_date = get_publication_date(title)
        print(f"Updating book ID {book_id} with publication date: {publication_date}")

        cursor.execute("UPDATE books SET publication_date = ? WHERE id = ?", (publication_date, book_id))
        conn.commit()

        time.sleep(1)  # 1 second delay

    conn.close()
    messagebox.showinfo("Success", "All publication dates have been updated successfully.")

# ToolTip Class for displaying helpful hints
class ToolTip:
    def __init__(self, widget, text):
        self.widget = widget
        self.text = text
        self.tooltip = None
        self.widget.bind("<Enter>", self.show_tooltip)
        self.widget.bind("<Leave>", self.hide_tooltip)

    def show_tooltip(self, event):
        x, y, _, _ = self.widget.bbox("insert")
        x += self.widget.winfo_rootx() + 25
        y += self.widget.winfo_rooty() + 25
        self.tooltip = tk.Toplevel(self.widget)
        self.tooltip.wm_overrideredirect(True)
        self.tooltip.wm_geometry(f"+{x}+{y}")
        label = tk.Label(self.tooltip, text=self.text, background="yellow", relief="solid")
        label.pack()

    def hide_tooltip(self, event):
        if self.tooltip:
            self.tooltip.destroy()
            self.tooltip = None

# Function to create and enhance the UI/UX layout
def create_ui():
    # Create main window
    root = tk.Tk()
    root.title("Book Database UI")
    root.geometry("600x400")
    
    # Create frames for different sections
    frame1 = tk.Frame(root, padx=10, pady=10)
    frame1.pack(padx=10, pady=10)

    frame2 = tk.Frame(root, padx=10, pady=10)
    frame2.pack(padx=10, pady=10)

    # Add labels and inputs for filters (example: Min Price)
    min_price_label = tk.Label(frame1, text="Min Price:")
    min_price_label.grid(row=0, column=0, padx=5, pady=5, sticky="w")
    min_price_input = tk.Entry(frame1)
    min_price_input.grid(row=0, column=1, padx=5, pady=5)

    # Add ToolTip to input field
    ToolTip(min_price_label, "Enter the minimum price to filter books.")

    # Add 'Fetch Data' button
    fetch_button = tk.Button(frame2, text="Fetch Data", command=update_publication_dates)
    fetch_button.grid(row=0, column=0, columnspan=2, pady=10)

    # Add ToolTip to the button
    ToolTip(fetch_button, "Click to update the publication dates of books.")

    # Add a success message on completion
    success_label = tk.Label(frame2, text="Ready to update books!", fg="green", font=("Arial", 12))
    success_label.grid(row=1, column=0, columnspan=2, pady=10)

    # Visual Enhancements
    fetch_button.config(font=("Arial", 12, "bold"), fg="white", bg="blue")
    
    # Run the UI main loop
    root.mainloop()

# Run the function to create UI
if __name__ == "__main__":
    create_ui()
A
