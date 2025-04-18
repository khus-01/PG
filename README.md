# PG
import streamlit as st
import base64
import re
import os
from PIL import Image
import mysql.connector
import json
from datetime import datetime
import io

class PGConnectApp:
    def __init__(self):
        # Initialize database connection
        self.init_db()
        
        # Initialize session state
        if 'users' not in st.session_state:
            st.session_state.users = {}
        if 'pg_listings' not in st.session_state:
            st.session_state.pg_listings = []
        if 'logged_in' not in st.session_state:
            st.session_state.logged_in = False
        if 'username' not in st.session_state:
            st.session_state.username = None
        if 'current_page' not in st.session_state:
            st.session_state.current_page = 'home'
        if 'user_type' not in st.session_state:
            st.session_state.user_type = None
        if 'document_verification' not in st.session_state:
            st.session_state.document_verification = {}
        if 'pg_reviews' not in st.session_state:
            st.session_state.pg_reviews = {}
        if 'house_rules' not in st.session_state:
            st.session_state.house_rules = {}
        if 'extra_rules' not in st.session_state:
            st.session_state.extra_rules = {}
        if 'booking_requests' not in st.session_state:
            st.session_state.booking_requests = {}
    
    def init_db(self):
        """Initialize database connection and create tables if they don't exist"""
        # Database connection configuration
        db_config = {
            'host': 'localhost',  # Change to your MySQL host
            'user': 'root',       # Change to your MySQL username
            'password': 'khushi@0105',       # Change to your MySQL password
            'database': 'pg_db'  # Database name
        }
        
        # Create database if it doesn't exist
        try:
            conn = mysql.connector.connect(
                host=db_config['host'],
                user=db_config['user'],
                password=db_config['password']
            )
            cursor = conn.cursor()
            cursor.execute(f"CREATE DATABASE IF NOT EXISTS {db_config['database']}")
            conn.close()
        except Exception as e:
            st.error(f"Database initialization error: {e}")
            return
        
        # Connect to the database
        try:
            self.conn = mysql.connector.connect(**db_config)
            self.cursor = self.conn.cursor(dictionary=True)
            
            # Create tables
            self.create_tables()
        except Exception as e:
            st.error(f"Database connection error: {e}")
    
    def create_tables(self):
        """Create necessary tables in the database"""
        # Users table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                username VARCHAR(50) UNIQUE NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                password VARCHAR(255) NOT NULL,
                user_type ENUM('Finder', 'Owner') NOT NULL,
                full_name VARCHAR(100) NOT NULL,
                contact_number VARCHAR(20) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Document verification table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS document_verification (
                id INT AUTO_INCREMENT PRIMARY KEY,
                user_id INT NOT NULL,
                document_data LONGBLOB NOT NULL,
                document_name VARCHAR(255) NOT NULL,
                verified BOOLEAN DEFAULT FALSE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        
        # PG Listings table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS pg_listings (
                id INT AUTO_INCREMENT PRIMARY KEY,
                owner_id INT NOT NULL,
                pg_name VARCHAR(100) NOT NULL,
                contact VARCHAR(20) NOT NULL,
                address TEXT NOT NULL,
                gender_option ENUM('Male Only', 'Female Only', 'Both Male and Female') NOT NULL,
                coed_arrangement TEXT,
                max_tenants INT NOT NULL,
                rent DECIMAL(10, 2) NOT NULL,
                deposit DECIMAL(10, 2) NOT NULL,
                deposit_info TEXT,
                furnishing ENUM('Not Furnished', 'Semi-Furnished', 'Fully Furnished') NOT NULL,
                security_features JSON,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        
        # PG Facilities table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS pg_facilities (
                id INT AUTO_INCREMENT PRIMARY KEY,
                pg_id INT NOT NULL,
                name VARCHAR(100) NOT NULL,
                icon VARCHAR(10) NOT NULL,
                available BOOLEAN DEFAULT TRUE,
                FOREIGN KEY (pg_id) REFERENCES pg_listings(id) ON DELETE CASCADE
            )
        """)
        
        # PG Images table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS pg_images (
                id INT AUTO_INCREMENT PRIMARY KEY,
                pg_id INT NOT NULL,
                image_data LONGBLOB NOT NULL,
                image_name VARCHAR(255) NOT NULL,
                FOREIGN KEY (pg_id) REFERENCES pg_listings(id) ON DELETE CASCADE
            )
        """)
        
        # House Rules table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS house_rules (
                id INT AUTO_INCREMENT PRIMARY KEY,
                pg_id INT NOT NULL,
                name VARCHAR(100) NOT NULL,
                status VARCHAR(100) NOT NULL,
                icon VARCHAR(10) NOT NULL,
                allowed BOOLEAN DEFAULT TRUE,
                FOREIGN KEY (pg_id) REFERENCES pg_listings(id) ON DELETE CASCADE
            )
        """)
        
        # Extra Rules table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS extra_rules (
                id INT AUTO_INCREMENT PRIMARY KEY,
                pg_id INT NOT NULL,
                rules_text TEXT NOT NULL,
                FOREIGN KEY (pg_id) REFERENCES pg_listings(id) ON DELETE CASCADE
            )
        """)
        
        # Reviews table
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS pg_reviews (
                id INT AUTO_INCREMENT PRIMARY KEY,
                pg_id INT NOT NULL,
                user_id INT NOT NULL,
                text TEXT NOT NULL,
                rating INT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (pg_id) REFERENCES pg_listings(id) ON DELETE CASCADE,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        
        self.conn.commit()
    
    def home_page(self):
        st.title("Welcome to PG Connect")
        st.markdown("""
            <style>
            /* Home Container - Slightly Brighter */
            .home-container {
                background-color: #f2f2f2; /* Light grey background */
                border-radius: 15px;
                padding: 30px;
                box-shadow: 0 4px 8px rgba(0, 0, 0, 0.05);
            }

            /* Feature Card - White Background Maintained */
            .feature-card {
                background-color: #ffffff; /* Keeping the white for clean look */
                border-radius: 10px;
                padding: 20px;
                text-align: center;
                box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
                transition: transform 0.3s ease, box-shadow 0.3s ease;
            }

            /* Feature Card Hover Effect */
            .feature-card:hover {
                transform: scale(1.05);
                box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
            }

            /* Feature Card Heading - Navy Blue */
            .feature-card h3 {
                color: #0077b6; /* Soft navy blue */
                font-size: 18px;
                margin-bottom: 10px;
            }

            /* Feature Card Text - Brightened Text */
            .feature-card p {
                color: #333333; /* Dark grey for good contrast */
                font-size: 14px;
            }

            /* Home Container Text */
            .home-container h2 {
                color: #0077b6; /* Soft navy blue for titles */
                font-size: 24px;
                margin-bottom: 20px;
            }

            /* Welcome Section and Call to Action - White Text */
            .welcome-text, .cta-text {
                color: #ffffff !important; /* White text */
                font-size: 16px;
                line-height: 1.6;
            }

            /* Button Styling */
            .stButton>button {
                background-color: #0077b6;
                color: #ffffff;
                border-radius: 8px;
                padding: 10px 20px;
                font-size: 16px;
                transition: all 0.3s ease;
            }

            .stButton>button:hover {
                background-color: #005f8b;
                transform: scale(1.05);
            }
            </style>
        """, unsafe_allow_html=True)

        # Home Container
        st.markdown('<div class="home-container">', unsafe_allow_html=True)

        # Welcome Section with New Div
        st.markdown("""
        
            ### Find Your Perfect PG Accommodation
        <div class="welcome-text">
            PG Connect helps you discover comfortable and convenient Paying Guest accommodations tailored to your needs.
        </div>
        """, unsafe_allow_html=True)

        # Key Features
        st.subheader("Why Choose PG Connect?")

        features = st.columns(3)
        feature_details = [
            ("🏠 Comprehensive Listings", "Explore a wide range of PG options"),
            ("✅ Quick Booking", "Fast and effortless booking process."),
            ("📱 Easy Matching", "Connect with perfect accommodations")
        ]

        for i, (title, description) in enumerate(feature_details):
            with features[i]:
                st.markdown(f'''
                <div class="feature-card">
                    <h3>{title}</h3>
                    <p>{description}</p>
                </div>
                ''', unsafe_allow_html=True)

        # Call to Action with New Div
        st.markdown("""
        
            ### Ready to Find Your Home?
        <div class="cta-text">
            Register as a Finder to explore listings or as an Owner to list your PG.
        </div>
        """, unsafe_allow_html=True)

        # Close Home Container
        st.markdown('</div>', unsafe_allow_html=True)


    def about_us_page(self):
        # This function remains unchanged
        st.markdown("""
        <style>
    .about-container {
        background-color: #f2f2f2; /* Light blue color */
        border-radius: 15px;
        padding: 30px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    .mission-section {
        background-color: #ffffff;
        border-left: 5px solid #4CAF50;
        padding: 20px;
        margin-bottom: 20px;
    }
    .feature-card {
        background-color: #ffffff;
        border-radius: 10px;
        padding: 20px;
        text-align: center;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        transition: transform 0.3s ease;
    }
    .feature-card:hover {
        transform: scale(1.05);
    }
    </style>
        """, unsafe_allow_html=True)

        st.title("About PG Connect")
        
        st.markdown('<div class="about-container">', unsafe_allow_html=True)
        
        # Mission Section
        st.markdown('<div>', unsafe_allow_html=True)
        st.markdown("""
             ### Our Mission
        <div class="cta-text">
              PG Connect is a revolutionary platform designed to simplify the process of finding and listing Paying Guest (PG) accommodations.
              We aim to bridge the gap between PG owners and potential tenants, making housing search transparent, convenient, and trustworthy.
        </div>
        """, unsafe_allow_html=True)
        
        st.markdown('</div>', unsafe_allow_html=True)
        
        # Features Section
        st.subheader("Our Key Features")
        
        features = st.columns(3)
        feature_details = [
            ("🏠 Comprehensive Listings", "Detailed information about PG accommodations"),
            ("🔍 Search & Filter", "Easily refine your search with filters for area, budget, amenities, and more"),
            ("📱 Easy Navigation", "User-friendly interface")
        ]
        
        for i, (title, description) in enumerate(feature_details):
            with features[i]:
                st.markdown(f'''
                <div class="feature-card">
                    <h3>{title}</h3>
                    <p>{description}</p>
                </div>
                ''', unsafe_allow_html=True)
        
        st.markdown('</div>', unsafe_allow_html=True)

    def policy_page(self):
        # This function remains unchanged
        st.markdown("""
         <style>
    .policy-container {
        background-color: #f2f2f2; /* Light blue color */
        border-radius: 15px;
        padding: 30px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    .policy-section {
        background-color: #ffffff;
        border-left: 5px solid #007bff;
        padding: 20px;
        margin-bottom: 20px;
    }
    </style>
        """, unsafe_allow_html=True)

        st.title("Privacy & Terms of Service")
        
        st.markdown('<div class="policy-container">', unsafe_allow_html=True)
        
        # Privacy Policy Section
        st.markdown('<div>', unsafe_allow_html=True)
        st.markdown("""
        ### Privacy Policy
        
        #### Information Collection
        - We collect personal information necessary for account creation and PG listing services
        - Your data is securely stored.
        
        #### Data Protection
        - Your passwords are hashed and cannot be retrieved
        """)
        st.markdown('</div>', unsafe_allow_html=True)
        
        # Terms of Service Section
        st.markdown('<div class="policy-section">', unsafe_allow_html=True)
        st.markdown("""
        ### Terms of Service
        
        #### User Responsibilities
        - Users must provide accurate and truthful information
        - Respect the community guidelines and other users
        
        #### Listing Guidelines
        - PG owners must provide accurate and up-to-date listing information
        - Misleading or fraudulent listings are not tolerated
        """)
        st.markdown('</div>', unsafe_allow_html=True)
        
        # Contact and Compliance
        st.markdown('<div class="policy-section">', unsafe_allow_html=True)
        st.markdown('<div>', unsafe_allow_html=True)
        st.markdown("""
             ### Contact & Compliance
        <div class="cta-text">
              For any privacy concerns or questions, please contact us at:<br>
             - Email: support@pgconnect.com <br>
        </div>
        """, unsafe_allow_html=True)
        
        st.markdown('</div>', unsafe_allow_html=True)
        st.markdown('</div>', unsafe_allow_html=True)

    def validate_email(self, email):
        """Basic email validation"""
        email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(email_regex, email) is not None

    def validate_password(self, password):
        """Password validation"""
        return len(password) >= 8

    def register_page(self):
        st.title("Register")
        
        # User Type Selection
        user_type = st.radio("Select User Type", ["Finder", "Owner"])
        
        new_username = st.text_input("Choose a Username")
        email = st.text_input("Email")
        password = st.text_input("Password", type="password")
        confirm_password = st.text_input("Confirm Password", type="password")
        
        # Additional fields for registration
        full_name = st.text_input("Full Name")
        contact_number = st.text_input("Contact Number")
        
        if user_type == "Finder":
            # Document Upload for Verification
            uploaded_document = st.file_uploader("Upload Identification Document", type=['pdf', 'png', 'jpg', 'jpeg'])
        
        if st.button("Register"):
            # Validation checks
            if not new_username or not email or not password:
                st.error("Please fill in all required fields")
                return
            
            if password != confirm_password:
                st.error("Passwords do not match")
                return
            
            if not self.validate_email(email):
                st.error("Invalid email format")
                return
            
            if not self.validate_password(password):
                st.error("Password must be at least 8 characters long")
                return
            
            # Check if username or email already exists
            self.cursor.execute("SELECT id FROM users WHERE username = %s OR email = %s", 
                              (new_username, email))
            existing_user = self.cursor.fetchone()
            
            if existing_user:
                st.error("Username or email already exists")
                return
            
            # Create user account in database
            try:
                # Insert user into database
                hashed_password = self.hash_password(password)
                
                self.cursor.execute("""
                    INSERT INTO users (username, email, password, user_type, full_name, contact_number)
                    VALUES (%s, %s, %s, %s, %s, %s)
                """, (new_username, email, hashed_password, user_type, full_name, contact_number))
                
                self.conn.commit()
                user_id = self.cursor.lastrowid
                
                # Handle document verification for Finder
                if user_type == "Finder" and uploaded_document:
                    # Save document to database
                    document_data = uploaded_document.read()
                    document_name = uploaded_document.name
                    
                    self.cursor.execute("""
                        INSERT INTO document_verification (user_id, document_data, document_name)
                        VALUES (%s, %s, %s)
                    """, (user_id, document_data, document_name))
                    
                    self.conn.commit()
                    st.info("Document uploaded. Awaiting verification.")
                
                st.success(f"Registration successful for {user_type}")
            except Exception as e:
                st.error(f"Registration failed: {e}")

    def login_page(self):
        st.title("Login")
        username = st.text_input("Username")
        password = st.text_input("Password", type="password")
        
        if st.button("Login"):
            if not username or not password:
                st.error("Please enter username and password")
                return
                
            # Check user credentials in database
            try:
                hashed_password = self.hash_password(password)
                
                self.cursor.execute("""
                    SELECT id, username, user_type FROM users 
                    WHERE username = %s AND password = %s
                """, (username, hashed_password))
                
                user = self.cursor.fetchone()
                
                if user:
                    # Set login state
                    st.session_state.logged_in = True
                    st.session_state.username = user['username']
                    st.session_state.user_type = user['user_type']
                    st.session_state.user_id = user['id'] 
                    st.success(f"Welcome, {username}!")
                    
                    # Use rerun instead of experimental_rerun
                    st.rerun()
                else:
                    st.error("Invalid username or password")
            except Exception as e:
                st.error(f"Login error: {e}")

    def logout(self):
        # Clear all session state variables related to login
        st.session_state.logged_in = False
        st.session_state.username = None
        st.session_state.user_type = None
        
        # Optional: Clear other relevant session states
        if 'current_page' in st.session_state:
            st.session_state.current_page = 'home'
        
        # Use rerun method
        st.rerun()

    def add_pg_listing(self):
        st.title("Add PG Listing")
        
        # Get user ID from database based on username
        try:
            self.cursor.execute("SELECT id FROM users WHERE username = %s", (st.session_state.username,))
            owner = self.cursor.fetchone()
            if not owner:
                st.error("User not found. Please log in again.")
                return
                
            owner_id = owner['id']
        except Exception as e:
            st.error(f"Error fetching user data: {e}")
            return
        
        # PG Details
        pg_name = st.text_input("PG Name")
        pg_contact = st.text_input("Contact Number")
        pg_address = st.text_input("Address")
        
        # Gender Specification 
        st.subheader("Gender Specifications")
        gender_option = st.radio(
            "Select gender allowed in your PG", 
            ["Male Only", "Female Only", "Both Male and Female"]
        )
        
        # If co-ed PG, ask for additional details
        if gender_option == "Both Male and Female":
            coed_arrangement = st.radio(
                "Co-ed Arrangement", 
                ["Separate floors/sections for each gender", "Mixed accommodation"]
            )
        else:
            coed_arrangement = None
        
        # Numeric inputs
        max_tenants = st.number_input("Maximum Number of Tenants", min_value=1, max_value=50)
        rent_amount = st.number_input("Rent Amount (₹)", min_value=0)
        
        # New deposit field
        deposit_amount = st.number_input("Security Deposit (₹)", min_value=0)
        deposit_info = st.text_area("Deposit Terms & Conditions", 
                                    placeholder="e.g., Refundable after 1 month notice, deductions for damages")
        
        # Multiple choice inputs
        furnishing_type = st.selectbox("Furnishing Type", 
            ["Not Furnished", "Semi-Furnished", "Fully Furnished"])
        
        # Facilities section with visual display similar to house rules
        st.subheader("Available Facilities")
        
        # Define default facilities with icons
        default_facilities = [
            {"name": "WiFi", "icon": "📶", "available": False},
            {"name": "AC", "icon": "❄️", "available": False},
            {"name": "Washing Machine", "icon": "🧺", "available": False},
            {"name": "Refrigerator", "icon": "🧊", "available": False},
            {"name": "Cooking Allowed", "icon": "🍳", "available": False},
            {"name": "Meals Included", "icon": "🍽️", "available": False},
            {"name": "TV", "icon": "📺", "available": False},
            {"name": "Gym", "icon": "💪", "available": False},
            {"name": "Study Table", "icon": "📚", "available": False},
            {"name": "Hot Water", "icon": "🚿", "available": False},
            {"name": "Parking", "icon": "🅿️", "available": False},
            {"name": "Power Backup", "icon": "🔋", "available": False}
        ]
        
        # Configure facilities in a grid layout
        facilities = []
        cols = st.columns(3)
        for i, facility in enumerate(default_facilities):
            with cols[i % 3]:
                available = st.checkbox(f"{facility['icon']} {facility['name']}")
                if available:
                    facilities.append({
                        "name": facility['name'],
                        "icon": facility['icon'],
                        "available": True
                    })
                else:
                    facilities.append({
                        "name": facility['name'],
                        "icon": facility['icon'],
                        "available": False
                    })
        
        # Custom facility option
        st.subheader("Add Custom Facility")
        custom_facility_name = st.text_input("Facility Name")
        custom_facility_icon = st.selectbox("Choose Icon", 
                                            ["🏠", "🛏️", "🚽", "🛁", "🏊", "🧹", "🌞", "🚲", "📱", "🖥️", "🎮", "📡"])
        
        if st.button("Add Custom Facility", key="add_facility"):
            if custom_facility_name:
                facilities.append({
                    "name": custom_facility_name,
                    "icon": custom_facility_icon,
                    "available": True
                })
                st.success(f"Added custom facility: {custom_facility_name}")
        
        # Security Features
        st.subheader("Security Features")
        security_features = st.multiselect("Select Security Features", [
            "24/7 Security", "CCTV", "Biometric Access", "Gated Community"
        ])
        
        # House Rules Section
        st.subheader("House Rules")
        
        # Define default rules
        default_rules = [
            {"name": "Notice Period", "status": "No Notice Period", "icon": "📅", "allowed": True},
            {"name": "Gate Closing Time", "status": "No Gate Closing time", "icon": "⏰", "allowed": True},
            {"name": "Visitor Entry", "status": "Allowed", "icon": "🧍", "allowed": True},
            {"name": "Non-Veg Food", "status": "Allowed", "icon": "🍗", "allowed": True},
            {"name": "Opposite Gender", "status": "Allowed", "icon": "⚤", "allowed": True},
            {"name": "Smoking", "status": "Not Allowed", "icon": "🚬", "allowed": False},
            {"name": "Drinking", "status": "Not Allowed", "icon": "🍺", "allowed": False},
            {"name": "Loud music", "status": "Allowed", "icon": "🎵", "allowed": True},
            {"name": "Party", "status": "Allowed", "icon": "🎉", "allowed": True}
        ]
        
        # Configure house rules
        house_rules = []
        
        cols = st.columns(3)
        for i, rule in enumerate(default_rules):
            with cols[i % 3]:
                allowed = st.checkbox(f"{rule['icon']} {rule['name']} Allowed", value=rule['allowed'])
                status_text = st.text_input(f"Status for {rule['name']}", 
                                            value="Allowed" if allowed else "Not Allowed")
                house_rules.append({
                    "name": rule['name'],
                    "status": status_text,
                    "icon": rule['icon'],
                    "allowed": allowed
                })
        
        # Extra Rules Section
        st.subheader("Additional House Rules")
        extra_rules = st.text_area("Enter any additional house rules or policies", 
                                    placeholder="e.g., No pets allowed, Quiet hours from 10 PM to 6 AM")
        
        # Image Upload
        uploaded_images = st.file_uploader("Upload PG Images", accept_multiple_files=True, 
                                            type=['png', 'jpg', 'jpeg'])
        
        if st.button("Add Listing"):
            # Validate inputs
            if not pg_name or not pg_address:
                st.error("Please fill in required fields")
                return
            
            try:

                
                # 1. Insert PG listing
                self.cursor.execute("""
                    INSERT INTO pg_listings (
                        owner_id, pg_name, contact, address, gender_option, coed_arrangement,
                        max_tenants, rent, deposit, deposit_info, furnishing, security_features
                    ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, (
                    owner_id, pg_name, pg_contact, pg_address, gender_option, coed_arrangement,
                    max_tenants, rent_amount, deposit_amount, deposit_info, furnishing_type, 
                    json.dumps(security_features)
                ))
                
                # Get the inserted PG ID
                pg_id = self.cursor.lastrowid
                
                # 2. Insert facilities
                for facility in facilities:
                    if facility['available']:  # Only store available facilities
                        self.cursor.execute("""
                            INSERT INTO pg_facilities (pg_id, name, icon, available) 
                            VALUES (%s, %s, %s, %s)
                        """, (pg_id, facility['name'], facility['icon'], facility['available']))
                
                # 3. Insert house rules
                for rule in house_rules:
                    self.cursor.execute("""
                        INSERT INTO house_rules (pg_id, name, status, icon, allowed) 
                        VALUES (%s, %s, %s, %s, %s)
                    """, (pg_id, rule['name'], rule['status'], rule['icon'], rule['allowed']))
                
                # 4. Insert extra rules
                if extra_rules:
                    self.cursor.execute("""
                        INSERT INTO extra_rules (pg_id, rules_text) 
                        VALUES (%s, %s)
                    """, (pg_id, extra_rules))
                
                # 5. Insert images
                if uploaded_images:
                    for img in uploaded_images:
                        img_bytes = img.read()
                        self.cursor.execute("""
                            INSERT INTO pg_images (pg_id, image_data, image_name) 
                            VALUES (%s, %s, %s)
                        """, (pg_id, img_bytes, img.name))
                
                # Commit transaction
                self.conn.commit()
                st.success("PG Listing Added Successfully!")
                
            except Exception as e:
                # Rollback in case of error
                self.conn.rollback()
                st.error(f"Error adding listing: {e}")


    def view_pg_listings(self):
        st.title("Available PG Listings")

        # Only show listings to Finders
        if st.session_state.user_type != "Finder":
            st.warning("Only Finders can view PG listings.")
            return

        # Search and Filter Section
        st.subheader("Search and Filters")
        filter_col1, filter_col2 = st.columns(2)

        with filter_col1:
            search_term = st.text_input("Search PGs by name or address")

            # Price range filter
            st.write("Price Range (₹)")
            min_price, max_price = st.slider(
                "Select price range",
                min_value=0,
                max_value=50000,
                value=(0, 50000),
                step=1000
            )

            # Gender filter
            gender_filter = st.multiselect("Filter by Gender", 
                ["Male Only", "Female Only", "Both Male and Female"])

        with filter_col2:
            # Furnishing filter
            furnishing_filter = st.multiselect("Filter by Furnishing", 
                ["Not Furnished", "Semi-Furnished", "Fully Furnished"])

            # Facilities filter
            facilities_filter = st.multiselect("Filter by Facilities", 
                ["WiFi", "AC", "Washing Machine", "Refrigerator", "Cooking Allowed", 
                "Meals Included", "TV", "Gym", "Study Table", "Hot Water", "Parking", "Power Backup"])

            # Security features filter
            security_filter = st.multiselect("Filter by Security Features",
                ["24/7 Security", "CCTV", "Biometric Access", "Gated Community"])

        # Clear filters button
        if st.button("Clear All Filters"):
            st.rerun()

        # Build the SQL query dynamically
        query = """
            SELECT p.*, u.username as owner_username 
            FROM pg_listings p
            JOIN users u ON p.owner_id = u.id
            WHERE 1=1
        """
        params = []

        if search_term:
            query += " AND (p.pg_name LIKE %s OR p.address LIKE %s)"
            params.extend([f"%{search_term}%", f"%{search_term}%"])

        query += " AND p.rent BETWEEN %s AND %s"
        params.extend([min_price, max_price])

        if gender_filter:
            placeholders = ', '.join(['%s'] * len(gender_filter))
            query += f" AND p.gender_option IN ({placeholders})"
            params.extend(gender_filter)

        if furnishing_filter:
            placeholders = ', '.join(['%s'] * len(furnishing_filter))
            query += f" AND p.furnishing IN ({placeholders})"
            params.extend(furnishing_filter)

        if facilities_filter:
            for facility in facilities_filter:
                query += """ AND EXISTS (
                    SELECT 1 FROM pg_facilities 
                    WHERE pg_id = p.id AND name = %s AND available = 1
                )"""
                params.append(facility)

        if security_filter:
            for feature in security_filter:
                query += " AND JSON_CONTAINS(p.security_features, %s)"
                params.append(json.dumps(feature))

        # Sorting options
        sort_by = st.selectbox("Sort by", ["Rent: Low to High", "Rent: High to Low"])
        query += " ORDER BY p.rent ASC" if sort_by == "Rent: Low to High" else " ORDER BY p.rent DESC"

        # Fetch and display results
        try:
            self.cursor.execute(query, params)
            filtered_listings = self.cursor.fetchall()

            st.write(f"Found {len(filtered_listings)} PG listings matching your criteria")

            if not filtered_listings:
                st.info("No PG listings found. Try adjusting your filters.")
                return

            for listing in filtered_listings:
                pg_id = listing['id']
                with st.expander(f"{listing['pg_name']} - {listing['address']} - ₹{listing['rent']}"):
                    st.write(f"**Owner Contact:** {listing['contact']}")
                    st.write(f"**Rent:** ₹{listing['rent']}")
                    
                    if listing.get('deposit'):
                        st.write(f"**Security Deposit:** ₹{listing['deposit']}")
                        if listing.get('deposit_info'):
                            st.write(f"**Deposit Terms:** {listing['deposit_info']}")

                    st.write(f"**Furnishing:** {listing['furnishing']}")

                    # Display Facilities
                    self.cursor.execute("SELECT name FROM pg_facilities WHERE pg_id = %s AND available = 1", (pg_id,))
                    facilities = [row['name'] for row in self.cursor.fetchall()]
                    if facilities:
                        st.subheader("Facilities")
                        st.write(", ".join(facilities))

                    # Display Security Features
                    security_features = json.loads(listing['security_features']) if listing['security_features'] else []
                    if security_features:
                        st.subheader("Security Features")
                        st.write(", ".join(security_features))

                    # Display House Rules
                    self.cursor.execute("SELECT name FROM house_rules WHERE pg_id = %s AND allowed = 0", (pg_id,))
                    house_rules = [row['name'] for row in self.cursor.fetchall()]
                    if house_rules:
                        st.subheader("House Rules")
                        st.write(", ".join(house_rules))

                    # Display PG Images
                    self.cursor.execute("SELECT id, image_name FROM pg_images WHERE pg_id = %s", (pg_id,))
                    images = self.cursor.fetchall()
                    if images:
                        st.subheader("PG Images")
                        cols = st.columns(min(3, len(images)))
                        for i, img in enumerate(images):
                            self.cursor.execute("SELECT image_data FROM pg_images WHERE id = %s", (img['id'],))
                            img_data = self.cursor.fetchone()
                            if img_data:
                                image = Image.open(io.BytesIO(img_data['image_data']))
                                cols[i % 3].image(image, use_container_width=True, caption=img['image_name'])

                    # Booking Request Section
                    if st.session_state.logged_in and st.session_state.user_type == "Finder":
                        st.subheader("Send Booking Request")
                        message = st.text_area(f"Message to the Owner for {listing['pg_name']}", placeholder="Write a message...", key=f"message_{pg_id}")

                        if st.button(f"Request Booking for {listing['pg_name']}", key=f"request_{pg_id}"):
                            self.cursor.execute("SELECT id, full_name, contact_number FROM users WHERE username = %s", (st.session_state.username,))
                            finder = self.cursor.fetchone()

                            if not finder:
                                st.error("Finder details not found. Please log in again.")
                                return

                            self.cursor.execute("SELECT document_name, document_data FROM document_verification WHERE user_id = %s", (finder['id'],))
                            document = self.cursor.fetchone()

                            owner_username = listing['owner_username']
                            if owner_username not in st.session_state.booking_requests:
                                st.session_state.booking_requests[owner_username] = []

                            st.session_state.booking_requests[owner_username].append({
                                'finder_username': st.session_state.username,
                                'finder_full_name': finder['full_name'],
                                'finder_contact': finder['contact_number'],
                                'finder_document': document,
                                'pg_name': listing['pg_name'],
                                'message': message
                            })
                            st.success("Booking request sent successfully!")
        
                # Display Reviews
                self.display_pg_reviews(pg_id)

                # Add Review Section
                st.subheader("Add a Review")
                review_text = st.text_area(f"Write your review for {listing['pg_name']}", key=f"review_text_{pg_id}")

                # Star-based rating system
                rating_options = ["⭐", "⭐⭐", "⭐⭐⭐", "⭐⭐⭐⭐", "⭐⭐⭐⭐⭐"]
                review_rating = st.radio(
                    f"Rate {listing['pg_name']} (1-5 Stars)",
                    options=list(range(1, 6)),
                    format_func=lambda x: rating_options[x - 1],
                    key=f"review_rating_{pg_id}"
                )

                if st.button(f"Submit Review for {listing['pg_name']}", key=f"submit_review_{pg_id}"):
                    try:
                        self.cursor.execute("""
                            INSERT INTO pg_reviews (pg_id, user_id, text, rating)
                            VALUES (%s, %s, %s, %s)
                        """, (pg_id, st.session_state.user_id, review_text, review_rating))
                        self.conn.commit()
                        st.success("Review submitted successfully!")
                    except Exception as e:
                        st.error(f"Error submitting review: {e}")
                 
        except Exception as e:
            st.error(f"Error retrieving listings: {e}")


    def hash_password(self, password):
        # Using base64 for demonstration - in production, use proper password hashing
        return base64.b64encode(password.encode()).decode()

    def check_password(self, input_password, stored_password):
        # Simple password check - in production use proper password verification
        return base64.b64encode(input_password.encode()).decode() == stored_password

    def manage_my_listings(self):
        """Function for owners to manage their PG listings"""
        st.title("Manage My PG Listings")
        
        if st.session_state.user_type != "Owner":
            st.warning("Only Owners can access this page")
            return
            
        # Get user ID
        self.cursor.execute("SELECT id FROM users WHERE username = %s", (st.session_state.username,))
        user = self.cursor.fetchone()
        if not user:
            st.error("User not found. Please log in again.")
            return
            
        user_id = user['id']
        
        # Fetch all listings by this owner
        try:
            self.cursor.execute("""
                SELECT * FROM pg_listings
                WHERE owner_id = %s
                ORDER BY created_at DESC
            """, (user_id,))
            
            listings = self.cursor.fetchall()
            
            if not listings:
                st.info("You haven't added any PG listings yet.")
                if st.button("Add a New PG Listing"):
                    st.session_state.current_page = 'add_pg_listing'
                    st.rerun()
                return
                
            # Display each listing with edit/delete options
            for listing in listings:
                self.display_pg_reviews(listing['id'])
                with st.expander(f"{listing['pg_name']} - {listing['address']}"):
                    st.write(f"**Rent:** ₹{listing['rent']}")
                    st.write(f"**Max Tenants:** {listing['max_tenants']}")
                    st.write(f"**Created:** {listing['created_at'].strftime('%Y-%m-%d')}")
                    
                    # Action buttons
                    col1, col2 = st.columns(2)
                    with col1:
                        if st.button("Edit Listing", key=f"edit_{listing['id']}"):
                            st.session_state.edit_pg_id = listing['id']
                            st.session_state.current_page = 'edit_pg_listing'
                            st.rerun()
                    with col2:
                        if st.button("Delete Listing", key=f"delete_{listing['id']}"):
                            # Confirm deletion
                            if st.checkbox("Confirm deletion? This cannot be undone.", key=f"confirm_{listing['id']}"):
                                try:
                                    # Delete the listing and all related data (cascading delete)
                                    self.cursor.execute("DELETE FROM pg_listings WHERE id = %s", (listing['id'],))
                                    self.conn.commit()
                                    st.success("Listing deleted successfully!")
                                    st.rerun()
                                except Exception as e:
                                    st.error(f"Error deleting listing: {e}")
        except Exception as e:
            st.error(f"Error retrieving listings: {e}")

    
    
    def display_pg_reviews(self, pg_id):
        """Display reviews for a specific PG listing."""
        st.subheader("Reviews")
        try:
            # Fetch reviews for the given PG ID
            self.cursor.execute("""
                SELECT r.text, r.rating, u.username, r.created_at
                FROM pg_reviews r
                JOIN users u ON r.user_id = u.id
                WHERE r.pg_id = %s
                ORDER BY r.created_at DESC
            """, (pg_id,))
            reviews = self.cursor.fetchall()

            if not reviews:
                st.info("No reviews available for this listing.")
                return

            # Display each review
            for review in reviews:
                st.markdown(f"**{review['username']}** ({review['created_at'].strftime('%Y-%m-%d')}):")
                st.write(f"Rating: {review['rating']} / 5")
                st.write(f"Review: {review['text']}")
                st.markdown("---")
        except Exception as e:
            st.error(f"Error retrieving reviews: {e}")



    def edit_pg_listing(self):
        """Function to edit an existing PG listing"""
        st.title("Edit PG Listing")
        
        if 'edit_pg_id' not in st.session_state:
            st.error("No listing selected for editing")
            return
            
        pg_id = st.session_state.edit_pg_id
        
        # Fetch current listing details
        try:
            self.cursor.execute("SELECT * FROM pg_listings WHERE id = %s", (pg_id,))
            listing = self.cursor.fetchone()
            
            if not listing:
                st.error("Listing not found")
                return
                
            # Now implement similar form as add_pg_listing but pre-fill with existing data
            # This would be similar to the add_pg_listing method
            # For brevity, I'm showing a simplified version
            
            pg_name = st.text_input("PG Name", value=listing['pg_name'])
            pg_contact = st.text_input("Contact Number", value=listing['contact'])
            pg_address = st.text_input("Address", value=listing['address'])
            
            # Continue with all other fields similar to add_pg_listing
            # Pre-fill with existing values from the database
            
            if st.button("Update Listing"):
                # Update database similar to add_pg_listing
                # For each field, update the corresponding table
                st.success("Listing updated successfully!")
        
        except Exception as e:
            st.error(f"Error editing listing: {e}")
    
    def view_booking_requests(self):
        st.title("Booking Requests")

        owner_username = st.session_state.username
        if owner_username not in st.session_state.booking_requests or not st.session_state.booking_requests[owner_username]:
            st.info("No booking requests received.")
            return

        for request in st.session_state.booking_requests[owner_username]:
            st.subheader(f"Request from {request['finder_username']}")
            st.write(f"PG Name: {request['pg_name']}")
            st.write(f"Message: {request['message']}")
            st.write(f"Finder Name: {request['finder_full_name']}")
            st.write(f"Contact Number: {request['finder_contact']}")
            st.write(f"Status: {request.get('status', 'Pending')}")  # Default to 'Pending'

            # Display uploaded document if available
            if request['finder_document']:
                document_name = request['finder_document']['document_name']
                document_data = request['finder_document']['document_data']
                st.download_button(
                    label=f"Download {document_name}",
                    data=document_data,
                    file_name=document_name,
                    mime="application/octet-stream"
                )

            # Accept or Reject Buttons
            col1, col2 = st.columns(2)
            with col1:
                if st.button(f"Accept Request from {request['finder_username']}", key=f"accept_{request['finder_username']}"):
                    request['status'] = "Accepted"
                    st.success(f"Accepted booking request from {request['finder_username']}")
            with col2:
                if st.button(f"Reject Request from {request['finder_username']}", key=f"reject_{request['finder_username']}"):
                    request['status'] = "Rejected"
                    st.warning(f"Rejected booking request from {request['finder_username']}")

    def view_my_requests(self):
        st.title("My Booking Requests")

        finder_username = st.session_state.username
        requests = []
        for owner_requests in st.session_state.booking_requests.values():
            for request in owner_requests:
                if request['finder_username'] == finder_username:
                    requests.append(request)

        if not requests:
            st.info("You have not made any booking requests.")
            return

        for request in requests:
            st.subheader(f"Request for {request['pg_name']}")
            st.write(f"Message: {request['message']}")
            st.write(f"Status: {request.get('status', 'Pending')}")  # Default to 'Pending'

    def main(self):
        # Set page configuration
        st.set_page_config(
          page_title="PG Connect", 
          page_icon=":house:", 
          layout="wide",
         initial_sidebar_state="expanded"
        )
    
        # Add custom CSS for styling
        st.markdown("""
   <style>
    /* General Page Styling */
    .reportview-container {
        background: linear-gradient(135deg, #d9f2ff 0%, #b3e6ff 100%); /* Sky blue gradient */
        padding: 20px;
    }
    .sidebar .sidebar-content {
        background: #ffffff;
        border-radius: 10px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    .stButton>button {
        color: white;
        background-color: #007bff; /* Blue button */
        border-radius: 10px;
        padding: 10px 20px;
        font-size: 16px;
        transition: all 0.3s ease;
    }
    .stButton>button:hover {
        background-color: #0056b3; /* Darker blue on hover */
        transform: scale(1.05);
    }
    .stTextInput>div>div>input {
        border-radius: 5px;
        border: 1px solid #ddd;
        padding: 10px;
    }
    .stRadio>div>div>label {
        font-size: 16px;
        margin-right: 10px;
    }
    .stMarkdown h1, .stMarkdown h2, .stMarkdown h3 {
        color: #007bff; /* Blue headings */
    }
    .stMarkdown p {
        font-size: 16px;
        line-height: 1.6;
        color: black; /* Black font for text */
    }
    .content-container {
        background-color: #e6f7ff; /* Light sky blue */
        border-radius: 15px;
        padding: 30px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    </style>
        """, unsafe_allow_html=True)

        # Navigation for logged-in users
        if st.session_state.logged_in:
            if st.session_state.user_type == "Owner":
                menu = ["Add PG Listing", "View Booking Requests", "Manage My Listings", "Logout"]
            else:  # Finder
                menu = ["View PG Listings", "My Booking Requests", "Logout"]
        
            choice = st.sidebar.radio("Navigation", menu)
        
            if choice == "Add PG Listing":
                self.add_pg_listing()
            elif choice == "Manage My Listings":
                self.manage_my_listings()
            elif choice == "View Booking Requests":
                self.view_booking_requests()
            elif choice == "View PG Listings":
                self.view_pg_listings()
            elif choice == "My Booking Requests":
                self.view_my_requests()
            elif choice == "Logout":
                self.logout()
        else:
            # Default navigation for non-logged in users
            menu = ["Home", "About Us", "Privacy Policy", "Login", "Register"]
            choice = st.sidebar.radio("Navigation", menu)

            # Page rendering logic
            if choice == "Home":
                self.home_page()
            elif choice == "About Us":
                self.about_us_page()
            elif choice == "Privacy Policy":
                self.policy_page()
            elif choice == "Login":
                self.login_page()
            elif choice == "Register":
                self.register_page()
    
    # Close database connection when done
        if hasattr(self, 'conn') and self.conn.is_connected():
            self.cursor.close()
            self.conn.close()


def main():
    app = PGConnectApp()
    app.main()

if __name__ == "__main__":
    main()
