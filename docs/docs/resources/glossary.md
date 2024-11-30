# Glossary 

This page has considerations about the structure of your program 
and considerations about how to troubleshoot the engineering 

## Considerations about logging modules 

```python
import datetime
import logging
from dateutil import parser
from typing import List, Optional, Dict

# Configure logging
logger = logging.getLogger('EventAnalyzer')
logger.setLevel(logging.DEBUG)
ch = logging.StreamHandler()
ch.setLevel(logging.DEBUG)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
ch.setFormatter(formatter)
logger.addHandler(ch)

DATE_FORMATS = [
    "%Y-%m-%d",
    "%d/%m/%Y",
    "%m/%d/%Y",
    "%d.%m.%Y",
    "%m.%d.%Y",
    "%b %d, %Y",
    "%B %d, %Y",
    "%d %B %Y",
    "%d %b %Y",
    "%Y-%m-%dT%H:%M:%S"
]

UNNAMED_EVENT = 'Unnamed Event'

def parse_date(
    date_str: str, 
    allow_fallback: bool = True, 
    timezone_aware: bool = False
) -> Optional[datetime.datetime]:
    """
    Attempt to parse a date string with multiple formats.

    Args:
        date_str (str): The date string to parse.
        allow_fallback (bool): Whether to allow fuzzy parsing as a fallback. Defaults to True.
        timezone_aware (bool): Whether to return a timezone-aware datetime object. Defaults to False.

    Returns:
        Optional[datetime.datetime]: The parsed date or None if parsing fails.
    """
    for fmt in DATE_FORMATS:
        try:
            parsed_date = datetime.datetime.strptime(date_str, fmt)
            if timezone_aware:
                parsed_date = parsed_date.replace(tzinfo=datetime.timezone.utc)
            logger.info(f"Parsed '{date_str}' successfully with format '{fmt}'.")
            return parsed_date
        except ValueError:
            logger.debug(f"Failed to parse '{date_str}' with format '{fmt}'.")

    if allow_fallback:
        try:
            parsed_date = parser.parse(date_str, fuzzy=True, dayfirst=True, yearfirst=True)
            if timezone_aware:
                parsed_date = parsed_date.replace(tzinfo=datetime.timezone.utc)
            logger.info(f"Parsed '{date_str}' with fuzzy matching.")
            return parsed_date
        except ValueError:
            logger.error(f"ValueError: Unable to parse date '{date_str}' with fuzzy matching.")
        except OverflowError:
            logger.error(f"OverflowError: Date '{date_str}' is out of range.")

    logger.warning(f"Returning None for unrecognized date: '{date_str}'")
    return None

def analyze_events(events: List[Dict[str, str]]) -> None:
    """
    Analyze and print events with parsed dates.

    Args:
        events (List[Dict[str, str]]): List of event dictionaries containing 'date' and 'event' keys.
    """
    if not events:
        logger.info("No events to analyze.")
        return

    invalid_events = []

    for event in events:
        date_str = event.get("date", "")
        event_date = parse_date(date_str)
        if event_date:
            event_name = event.get('event', UNNAMED_EVENT)
            formatted_date = event_date.strftime('%A, %B %d, %Y')
            print(f"Event: {event_name} on {formatted_date}")
        else:
            logger.warning(f"Skipping event due to invalid date: {event}")
            invalid_events.append({"event": event, "reason": f"Invalid date format: '{date_str}'"})

    if invalid_events:
        logger.info(f"Invalid events: {invalid_events}")

if __name__ == "__main__":
    events_input = [
        {"date": "2023-01-01", "event": "New Year's Day"},
        {"date": "2023-02-14", "event": "Valentine's Day"},
        {"date": "2023-07-04", "event": "Independence Day"},
        {"date": "2023-12-25", "event": "Christmas Day"},
        {"date": "Jul 4, 2023", "event": "Independence Day"},
        {"date": "2023-07-04 00:00:00", "event": "Midnight Independence Day Celebration"},
        {"date": "4 July 2023", "event": "Independence Day"},
        {"date": "July 4th, 2023", "event": "Independence Day"},
        {"date": "2023-07-04T00:00:00", "event": "Independence Day"},
        {"date": "4/7/23", "event": "Independence Day"},
        {"date": "4.7.23", "event": "Independence Day"},
        {"date": "2023-07-04", "event": "Independence Day"}  # Invalid date format
    ]

    analyze_events(events_input)
```

## Considerations about events


```python
events = [
    {"event": "Conference", "date": "2024-10-18"},
    {"event": "Meetup", "date": "18/10/2024"},
    {"event": "Hackathon", "date": "InvalidDate"}
]

analyze_events(events)
```

## Logging handlers

```python
from logging.handlers import RotatingFileHandler


class CustomLoggingHandler:
    def __init__(
        self,
        log_name: str = "EventAnalyzer",
        log_file: Optional[str] = None,
        log_level: int = logging.DEBUG,
        max_bytes: int = 10 * 1024 * 1024,  # 10 MB
        backup_count: int = 5,
        formatter: Optional[logging.Formatter] = None,
        handlers: Optional[list] = None
    ):
        self.logger = logging.getLogger(log_name)
        self.logger.setLevel(log_level)

        # Use provided formatter or create a default one
        self.formatter = formatter or self.create_formatter()

        # Create and add handlers
        if handlers:
            for handler in handlers:
                handler.setFormatter(self.formatter)
                self.logger.addHandler(handler)
        else:
            self.add_default_handlers(log_file, log_level, max_bytes, backup_count)

    def create_formatter(self) -> logging.Formatter:
        """Creates a default logging formatter."""
        return logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")

    def create_file_handler(
        self, log_file: str, log_level: int, max_bytes: int, backup_count: int
    ) -> Optional[logging.Handler]:
        """Creates a RotatingFileHandler."""
        try:
            file_handler = RotatingFileHandler(
                log_file, maxBytes=max_bytes, backupCount=backup_count
            )
            file_handler.setLevel(log_level)
            file_handler.setFormatter(self.formatter)
            return file_handler
        except (IOError, PermissionError) as e:
            self.logger.error(f"Failed to set up file handler: {e}")
            return None

    def add_default_handlers(
        self, log_file: Optional[str], log_level: int, max_bytes: int, backup_count: int
    ):
        """Adds default handlers to the logger."""
        # Stream Handler (Console output)
        stream_handler = logging.StreamHandler()
        stream_handler.setLevel(log_level)
        stream_handler.setFormatter(self.formatter)
        self.logger.addHandler(stream_handler)

        # Rotating File Handler (File output with rotation)
        if log_file:
            file_handler = self.create_file_handler(log_file, log_level, max_bytes, backup_count)
            if file_handler:
                self.logger.addHandler(file_handler)

    def log(self, level: int, message: str):
        """Logs a message at the specified log level."""
        log_method = getattr(self.logger, logging.getLevelName(level).lower(), None)
        if callable(log_method):
            log_method(message)
        else:
            self.logger.error(f"Invalid log level: {level}")

    def get_logger(self) -> logging.Logger:
        """Returns the configured logger instance."""
        return self.logger


# Usage example
if __name__ == "__main__":
    custom_formatter = logging.Formatter("%(levelname)s: %(message)s")
    custom_handlers = [logging.StreamHandler()]

    log_handler = CustomLoggingHandler(
        log_file="event_analyzer.log",
        formatter=custom_formatter,
        handlers=custom_handlers
    )

    # Log some events at different levels
    log_handler.log(logging.INFO, "This is an info message.")
    log_handler.log(logging.WARNING, "This is a warning message.")
    log_handler.log(logging.ERROR, "This is an error message.")
    log_handler.log(logging.DEBUG, "This is a debug message.")
```

## Box Plot

```python
# boxplot.py

import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns



def plot_boxplot(
    dataframe: pd.DataFrame, 
    x_column: str, 
    y_column: str, 
    color: str = 'blue', 
    title: Optional[str] = None, 
    figsize: Tuple[int, int] = (8, 6)
) -> Optional[plt.Figure]:
    """
    Plot boxplot.

    Parameters:
        dataframe (pd.DataFrame): Input DataFrame.
        x_column (str): Name of the column for x-axis.
        y_column (str): Name of the column for y-axis.
        color (str, optional): Color of the boxplot.
        title (str, optional): Title of the plot.
        figsize (tuple, optional): Figure size.

    Returns:
        Optional[plt.Figure]: The matplotlib figure object if successful, None otherwise.
    """
    fig, ax = plt.subplots(figsize=figsize)
    try:
        if dataframe.empty:
            raise ValueError("DataFrame is empty.")

        if not all(col in dataframe.columns for col in [x_column, y_column]):
            raise ValueError(f"Columns '{x_column}' or '{y_column}' not found in DataFrame.")

        dataframe = dataframe.dropna(subset=[x_column, y_column])
        sns.boxplot(data=dataframe, x=x_column, y=y_column, color=color, ax=ax)
        ax.set(
            title=f"Boxplot of {y_column} grouped by {x_column}" if title is None else title,
            xlabel=x_column, 
            ylabel=y_column
        )
        return fig  # Return the figure object
    except ValueError as ve:
        logger.error(f"Error plotting boxplot: {ve}")
        return None

def test_boxplot() -> None:
    """Test the plot_boxplot function."""
    data = {'group': ['A', 'A', 'B', 'B', 'B'], 'value': [1, 2, 3, 4, 5]}
    df = pd.DataFrame(data)
    fig = plot_boxplot(df, 'group', 'value')
    if fig:
        plt.show()

if __name__ == "__main__":
    test_boxplot()
```
## Compare Dates

```python
def validate_date_columns(dataframe: pd.DataFrame, required_columns: Optional[list] = None) -> None:
    """Ensure the required columns exist in the DataFrame."""
    required_columns = required_columns or ["start_date", "end_date"]
    missing_columns = [col for col in required_columns if col not in dataframe.columns]

    if missing_columns:
        raise ValueError(f"Missing required columns: {', '.join(missing_columns)}")

def convert_to_datetime(dataframe: pd.DataFrame, columns: list[str]) -> pd.DataFrame:
    """Convert specified columns to datetime."""
    for column in columns:
        dataframe[column] = pd.to_datetime(dataframe[column], errors='raise')
    return dataframe

def compare_dates(dataframe: pd.DataFrame) -> pd.DataFrame:
    """
    Compare start and end dates for each row in the DataFrame.

    Parameters:
        dataframe (pd.DataFrame): The DataFrame containing the data to be analyzed.

    Returns:
        pd.DataFrame: A new DataFrame containing the original data along with a new column
                      indicating whether the end date is after the start date.
    """
    try:
        # Validate and convert columns
        validate_date_columns(dataframe)
        dataframe = convert_to_datetime(dataframe, ["start_date", "end_date"])

        # Add comparison column
        dataframe["end_after_start"] = dataframe["end_date"] > dataframe["start_date"]
        return dataframe

    except (ValueError, pd.errors.ParserError) as e:
        logger.error(f"Error in compare_dates: {e}")
        raise

def example_usage() -> None:
    """Example usage of the compare_dates function."""
    data = {
        "id": [1, 2, 3],
        "start_date": ["2024-01-01", "2024-02-01", "2024-03-01"],
        "end_date": ["2024-01-15", "2024-02-10", "2024-03-20"],
    }
    df = pd.DataFrame(data)

    try:
        result_df = compare_dates(df)
        print(result_df)
    except ValueError as e:
        logger.error(f"Example usage failed: {e}")

if __name__ == "__main__":
    example_usage()
```

## Data Processor

```python
import re


class DataProcessor:
    def __init__(self):
        self.email_pattern = re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b')
        self.word_pattern = re.compile(r'\b[a-zA-Z]{5,15}\b')
        self.url_pattern = re.compile(r'(https?:\/\/)?([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*)*\/?$')
        self.phone_pattern = re.compile(r'(\+\d{1,2}\s)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}')

    def find_emails(self, text: str) -> List[str]:
        """Find all email addresses in the given text."""
        return self.email_pattern.findall(text.lower())

    def find_words(self, text: str) -> List[str]:
        """Find all words of length 5 to 15 in the given text."""
        return self.word_pattern.findall(text.lower())

    def find_urls(self, text: str) -> List[str]:
        """Find all URLs in the given text."""
        return self.url_pattern.findall(text.lower())

    def find_phone_numbers(self, text: str) -> List[str]:
        """Find all phone numbers in the given text."""
        return self.phone_pattern.findall(text)

    def count_word_occurrences(self, text: str) -> Dict[str, int]:
        """Count occurrences of each word in the given text."""
        words = self.word_pattern.findall(text.lower())
        return {word: words.count(word) for word in set(words)}

    def extract_domains(self, emails: List[str]) -> List[str]:
        """Extract domain names from email addresses."""
        return [re.search(r'@(.+)$', email).group(1) for email in emails]

    def validate_input(self, text: str):
        """Validate input text."""
        if not isinstance(text, str):
            raise ValueError("Input must be a string.")
        if len(text.strip()) == 0:
            raise ValueError("Input cannot be empty.")

# Example usage
if __name__ == "__main__":
    processor = DataProcessor()

    sample_text = """
    Here are some sample emails: test@example.com, hello@world.com.
    Some words: example, processor, text, data.
    Some URLs: https://example.com, http://test.com.
    Phone numbers: +1 (123) 456-7890, 0987654321
    """

    try:
        processor.validate_input(sample_text)

        emails = processor.find_emails(sample_text)
        words = processor.find_words(sample_text)
        urls = processor.find_urls(sample_text)
        phone_numbers = processor.find_phone_numbers(sample_text)
        word_counts = processor.count_word_occurrences(sample_text)
        domains = processor.extract_domains(emails)

        print("Emails found:", emails)
        print("Words found:", words)
        print("URLs found:", urls)
        print("Phone numbers found:", phone_numbers)
        print("Word occurrences:", word_counts)
        print("Domains extracted:", domains)

    except ValueError as e:
        print(f"Error: {e}")
```
