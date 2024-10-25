# Linkden-goggle-scholar-extraction-graphs

!pip install pdfminer.six pandas pdfplumber

import re
import pandas as pd
from pdfminer.high_level import extract_text
from google.colab import files

# Initialize DataFrame to store extracted information
columns = ['File Name', 'Bachelor Degree', 'Bachelor School', 'Bachelor Country', 'PhD Degree', 'PhD Year']
df = pd.DataFrame(columns=columns)

# Define section names to look for
SECTIONS = ["Education", "Academic Degrees"]

# Updated patterns for degree extraction with more flexibility
DEGREE_PATTERNS = {
    'Bachelor': r"(?i)(B\.?A\.?|B\.?S\.?|B\.?E\.?|B\.?Sc\.?|A\.?B\.?|B\.?C\.?|Bachelor(?:'s)?|Bachelors?|Bachelor (?:of|in) [A-Za-z]+|Bachelor (?:of|in) Science.*?(\d{4})|M\.?Sc|MSc|MSc(?:\.)?|sb(?:\.?)?)",
    'PhD': r"(?i)(Ph\.?D|Doctor of Philosophy|Doctorate|Doctoral)"
}

# Function to extract text from PDF
def extract_text_from_pdf(pdf_path):
    return extract_text(pdf_path)

# Function to extract relevant section from the CV text
def get_relevant_section(text, section_name):
    pattern = re.compile(rf'{section_name}', re.IGNORECASE)
    lines = text.split('\n')

    start_index = None
    end_index = None

    for i, line in enumerate(lines):
        if pattern.search(line):
            start_index = i
            for j in range(i + 1, len(lines)):
                if re.match(r'^\s*(?=HONORS|SERVICE|CAREER|EDITORIAL|WORKS|PUBLICATIONS|POSITIONS|EXPERIENCE|RESEARCH INTERESTS)', lines[j], re.IGNORECASE):
                    end_index = j
                    break
            if end_index is None:
                end_index = len(lines)
            break

    if start_index is not None:
        return '\n'.join(lines[start_index:end_index])
    return ""

# Function to extract degrees (Bachelor or PhD) with enhanced flexibility
def extract_degrees(text, degree_type):
    lines = text.split('\n')
    degrees = []

    for i, line in enumerate(lines):
        if "award" not in line.lower():  # Ensure "award" is not in the line
            if degree_type == 'Bachelor' and re.search(DEGREE_PATTERNS['Bachelor'], line):
                # Look for any 4-digit number that might be the year
                year_match = re.search(r'(\d{4})', line)
                degree_line = line.strip()
                if year_match:
                    year = year_match.group(1)
                    degrees.append((int(year), degree_line))  # Store year as integer for sorting
                else:
                    degrees.append((float('inf'), degree_line))  # Use infinity if year is unknown
            elif degree_type == 'PhD' and re.search(DEGREE_PATTERNS['PhD'], line):
                degrees.append((i, line.strip()))

    return degrees

# Function to extract Bachelor School and Country
def extract_bachelor_school_country(text):
    lines = text.split('\n')
    school = ""
    country = ""

    for line in lines:
        # Check if line mentions 'school' or 'university'
        if 'school' in line.lower() or 'university' in line.lower():
            school = line.strip()
            # Now look for a country (assumes it's in the same line or in the next line)
            if 'united states' in line.lower():
                country = 'United States'
            elif 'canada' in line.lower():
                country = 'Canada'
            elif 'uk' in line.lower() or 'united kingdom' in line.lower():
                country = 'United Kingdom'
            # Add more countries as needed

    return school, country

# Function to extract PhD year
def extract_phd_info(text):
    lines = text.split('\n')
    phd_year = None

    for i, line in enumerate(lines):
        phd_match = re.search(r"(?i)(Ph\.?D|Doctorate|Doctoral)", line)
        if phd_match:
            # First check the current line for a year
            year_match = re.search(r'(\d{4})', line)
            if year_match:
                phd_year = year_match.group(1)
                break
            # If not found, check the next line
            if i + 1 < len(lines):
                year_match_next = re.search(r'(\d{4})', lines[i + 1])
                if year_match_next:
                    phd_year = year_match_next.group(1)
                    break

    return phd_year

# Main process
if __name__ == '__main__':
    while True:
        # Upload the file
        uploaded = files.upload()

        # Get the file path (after upload, the filename is the key)
        pdf_path = list(uploaded.keys())[0]

        # Extract text from the uploaded PDF
        text = extract_text_from_pdf(pdf_path)

        # Extract relevant sections for EDUCATION and ACADEMIC DEGREES
        education_text = get_relevant_section(text, 'Education')
        academic_degrees_text = get_relevant_section(text, 'Academic Degrees')

        # Initialize containers for results
        bachelor_degrees = []
        phd_degrees = []

        # Process the EDUCATION section
        if education_text:
            print("Extracting from EDUCATION section...")
            bachelor_degrees = extract_degrees(education_text, 'Bachelor')
            phd_degrees = extract_degrees(education_text, 'PhD')

            # Extract Bachelor School and Country
            bachelor_school, bachelor_country = extract_bachelor_school_country(education_text)
        else:
            bachelor_school, bachelor_country = "", ""

        # Process the ACADEMIC DEGREES section if not found in EDUCATION
        if academic_degrees_text:
            print("Extracting from ACADEMIC DEGREES section...")
            bachelor_degrees += extract_degrees(academic_degrees_text, 'Bachelor')
            phd_degrees += extract_degrees(academic_degrees_text, 'PhD')

        # Extract PhD year
        phd_year = extract_phd_info(education_text) if education_text else None
        if not phd_year and academic_degrees_text:
            phd_year = extract_phd_info(academic_degrees_text)

        # Select the earliest Bachelor degree found
        bachelor_info = ""
        if bachelor_degrees:
            # Sort by year to find the earliest one
            bachelor_degrees.sort(key=lambda x: x[0])  # Sort by year
            bachelor_info = bachelor_degrees[0][1]  # Get the degree line of the first one

        # Prepare PhD information
        phd_info = ', '.join([degree for _, degree in phd_degrees]) or "PhD not found"

        # Create a DataFrame for the current file
        df_new = pd.DataFrame([{
            'File Name': pdf_path,
            'Bachelor Degree': bachelor_info if bachelor_info else "Bachelor's Degree not found",
            'Bachelor School': bachelor_school if bachelor_school else "Bachelor's School not found",
            'Bachelor Country': bachelor_country if bachelor_country else "Country not found",
            'PhD Degree': phd_info,
            'PhD Year': phd_year if phd_year else 'PhD year not found'
        }])

        # Update the existing DataFrame
        df = pd.concat([df, df_new], ignore_index=True)

        # Print the updated DataFrame
        print(df)

        # Ask if the user wants to continue
        continue_choice = input("Do you want to process another CV? (yes/no): ").strip().lower()
        if continue_choice != 'yes':
            print("Processing complete.")
            break
            
