# budget-bakers-wallet-export
Web Scraping Wallet Records
This script is designed to scrape wallet records from a webpage, extract details such as category, account, description, amount, date, and currency, and then save the data into a CSV file.

> [!CAUTION] 
> This JavaScript code is for **research purposes** only and is not intended to be used as a basis for any commercial or non-research purposes.  
> The use of this code is at the user's own risk, and the author(s) assume no responsibility for any misuse or unintended consequences.  
>
> **By using this code, the user acknowledges and agrees to the following:**
> 
> - The user is solely responsible for any legal consequences arising from the use of this code.  
> - The user will not hold the author(s) liable for any damages, losses, or legal actions resulting from the use of this code.  
> - The user agrees to comply with any applicable laws and regulations, including but not limited to copyright, intellectual property, and privacy laws.  
> - Furthermore, the user acknowledges that using this code may violate WhatsApp's Terms of Service and could potentially lead to account bans or other penalties.  
> - The user is advised to consult their own legal counsel before using this code to ensure compliance with all applicable laws and regulations.  
## Usage
1. Open [https://web.budgetbakers.com/records](https://web.budgetbakers.com/records) in your browser.
2. Select "all" in the filters
3. Press F12 to open the developer tools.
4. Go to the `console` tab.
5. Paste the script.
```js
// CSS Selectors remain the same
const SELECTORS = {
    RECORDS: '._3wwqabSSUyshePYhPywONa > ._3oJhqSCX8H5S0i6pA59f9k',
    DATES: '.MhNEgOnlVNRo3U-Ti1ZHP',
    SECTION_SELECTOR: '.VypTY5DQ_tmahm5VdHFJK > div:not([class])',
    CLASSES: {
        category: '._1Q3dkM4Dh6bjIxICrBMsvZ',
        account: '.haztXbqN9W6_Sa8X7ZjHh',
        desc: '.qcICMAjXBzU_8kvhCu6Ir',
        labels: '._2yWsrOsWf0KGrXIxhhDI2I > ._3qB8ZxU3QZU1r0vgpc39K_',
        amount: '.zh61r2aVULGpP5-PnSAHX > ._2incM6fyIxbGkeydtYoltF'
    }
};

// Utility functions
const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

const cleanDate = dateStr => {
    try {
        dateStr = dateStr.trim();
        
        // Handle special cases for relative dates
        const now = new Date();
        if (dateStr.toLowerCase() === "yesterday") {
            const yesterday = new Date(now);
            yesterday.setDate(now.getDate() - 1);
            return yesterday.toISOString().split('T')[0];
        } else if (dateStr.toLowerCase() === 'today'){
            return now.toISOString().split('T')[0];
        }

        // Parse "Month Day" format, e.g., "November 6"
        const monthDayMatch = dateStr.match(/^(January|February|March|April|May|June|July|August|September|October|November|December) (\d{1,2})$/);
        if (monthDayMatch) {
            const monthName = monthDayMatch[1];
            const day = parseInt(monthDayMatch[2]);

            const months = {
                'January': 0, 'February': 1, 'March': 2, 'April': 3,
                'May': 4, 'June': 5, 'July': 6, 'August': 7,
                'September': 8, 'October': 9, 'November': 10, 'December': 11
            };

            const month = months[monthName];
            const year = now.getFullYear();
            const date = new Date(year, month, day);

            return date.toISOString().split('T')[0];
        }

        // Fallback: try direct parsing
        const date = new Date(dateStr);
        if (!isNaN(date.getTime())) {
            return date.toISOString().split('T')[0];
        } else {
            console.error(`Invalid date string: ${dateStr}`);
            return dateStr; // Return original string if parsing fails
        }
    } catch (error) {
        console.error(`Error cleaning date: ${dateStr}`, error);
        return dateStr; // Return original string if any error occurs
    }
};


const cleanAmount = amountStr => {
    // Define common currency symbols and their respective codes
    const currencySymbols = {
        '$': 'USD',
        '€': 'EUR',
        '£': 'GBP',
        '¥': 'JPY'
    };

    // Regex to detect symbol and numeric value
    const symbolMatch = amountStr.match(/^[\$\€\£\¥]|USD|EUR|GBP|JPY/);
    const currency = symbolMatch ? currencySymbols[symbolMatch[0]] || symbolMatch[0] : 'USD'; // Default to USD if not detected

    // Remove non-numeric characters and parse amount
    const numericAmount = parseFloat(amountStr.replace(/[^\d.-]/g, ''));

    return { amount: numericAmount, currency };
};

const getRecordDetails = record => {
    const category = record.querySelector(SELECTORS.CLASSES.category)?.textContent.trim() || '';
    const account = record.querySelector(SELECTORS.CLASSES.account)?.textContent.trim() || '';
    const description = record.querySelector(SELECTORS.CLASSES.desc)?.textContent.trim() || '';
    const labelsElements = record.querySelectorAll(SELECTORS.CLASSES.labels);
    const labels = Array.from(labelsElements).map(label => label.textContent.trim());
    
    const amountText = record.querySelector(SELECTORS.CLASSES.amount)?.textContent.trim();
    const { amount, currency } = cleanAmount(amountText);

    return { category, account, description, labels, amount, currency };
};

const getDates = () => {
    return Array.from(document.querySelectorAll(SELECTORS.DATES))
        .map(date => date.textContent.trim());
};


const getTuplesList = () => {
    const tuples = [];
    let currentIndex = 0;
    
    // Get all section elements for date-based grouping
    document.querySelectorAll(SELECTORS.SECTION_SELECTOR).forEach(section => {
        const recordsCount = section.querySelectorAll(SELECTORS.RECORDS).length;
        tuples.push([currentIndex, recordsCount]);
        currentIndex += recordsCount;
    });
    return tuples;
};


// Main scraping function
async function scrapeWallet() {
    try {
        // Scroll to load all records
        let oldDatesCount, newDatesCount;
        document.documentElement.scrollTop = 0;
        await delay(2000);
        do {
            oldDatesCount = getDates().length;
            window.scrollTo(0, document.body.scrollHeight);
            await delay(1000);
            newDatesCount = getDates().length;
        } while (oldDatesCount !== newDatesCount);
        
        // Get all records, dates, and tuples
        const webRecords = document.querySelectorAll(SELECTORS.RECORDS);
        const records = Array.from(webRecords).map(getRecordDetails);
        const dates = getDates().map(cleanDate);
        const tuples = getTuplesList();
        
        // Process records based on section boundaries
        const allRecords = [];
        
        tuples.forEach(([startIndex, count], sectionIndex) => {
            const sectionDate = dates[sectionIndex]; // Date for this section

            for (let i = startIndex; i < startIndex + count; i++) {
                if (i >= records.length) break;
                
                const { category, account, description, labels, amount, currency } = records[i];
                const type = amount < 0 ? 'Expense' : 'Income';

                allRecords.push({
                    date: sectionDate,
                    type,
                    category,
                    account,
                    description,
                    label: labels,
                    currency,
                    amount
                });
            }
        });
        
        // Download as CSV
        const csv = convertToCSV(allRecords);
        downloadCSV(csv, 'wallet_records.csv');
        
        return allRecords;
    } catch (error) {
        console.error('Error scraping wallet:', error);
        throw error;
    }
}



// Helper functions for CSV export
function convertToCSV(records) {
    const headers = Object.keys(records[0]);
    const rows = records.map(record => 
        headers.map(header => {
            const value = record[header];
            // Handle arrays (like labels) and other types appropriately
            return JSON.stringify(Array.isArray(value) ? value.join(';') : value);
        }).join(',')
    );
    
    return [
        headers.join(','),
        ...rows
    ].join('\n');
}

function downloadCSV(csv, filename) {
    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const link = document.createElement('a');
    const url = URL.createObjectURL(blob);
    link.setAttribute('href', url);
    link.setAttribute('download', filename);
    link.style.visibility = 'hidden';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}

await scrapeWallet();
```
