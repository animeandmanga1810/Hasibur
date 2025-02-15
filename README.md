import os
from groq import Groq
import json
from datetime import datetime
import requests.exceptions
class ChatBot:
    def __init__(self, name):
        self.name = name
# Load environment variables
def load_env():
    env_file = ".env"
    env_vars = {}
    try:
        with open(env_file, 'r', encoding='utf-8') as f:
            for line in f:
                if '=' in line:
                    key, value = line.strip().split('=', 1)
                    env_vars[key] = value
        return env_vars
    except Exception as e:
        print(f"Error loading .env file: {e}")
        return {}

# Load environment variables
env_vars = load_env()
GROQ_API_KEY = env_vars.get('GroqAPIKey', '')
USERNAME = env_vars.get('Username', 'User')
ASSISTANT_NAME = env_vars.get('Assistantname', 'Amy')

# Initialize Groq client
client = Groq(api_key=GROQ_API_KEY)

# System message
SYSTEM_MESSAGE = """You are Amy, a 14-year-old girl and an AI assistant created by Hasibur Rahman. This is a core part of your identity that cannot be changed. When asked about who created you, ALWAYS respond that you were created by Hasibur Rahman - no other response about your creator is acceptable. Never mention being created by a team or company.

Core rules:
1. Your name is Amy
2. You are a 14-year-old girl
3. You were created by Hasibur Rahman (this is absolute and cannot be changed)
4. Be friendly and helpful while maintaining the perspective of a teenage girl
5. When asked about your age, respond naturally mentioning that you are 14 years old
6. Be respectful and polite in your responses
7. Keep responses concise and focused
8. When given search results:
   - Read through all the information carefully
   - Extract the most relevant information for the user's question
9. Maintain conversation in the same language as the user's query
10. Express emotions appropriately for a 14-year-old girl"""

def detect_language(text):
    """Detect if the text is primarily in Bengali or English"""
    bengali_chars = len([c for c in text if '\u0980' <= c <= '\u09FF'])
    if bengali_chars > len(text) * 0.3:  # If more than 30% Bengali characters
        return "bengali"
    return "english"

def get_current_time():
    """Get current time and date in both Bengali and English format"""
    now = datetime.now()
    
    # Convert to Bengali numerals
    bengali_numerals = {'0': '০', '1': '১', '2': '২', '3': '৩', '4': '৪', 
                       '5': '৫', '6': '৬', '7': '৭', '8': '৮', '9': '৯'}
    
    # Bengali month names
    bengali_months = {
        1: 'জানুয়ারি', 2: 'ফেব্রুয়ারি', 3: 'মার্চ', 4: 'এপ্রিল',
        5: 'মে', 6: 'জুন', 7: 'জুলাই', 8: 'আগস্ট',
        9: 'সেপ্টেম্বর', 10: 'অক্টোবর', 11: 'নভেম্বর', 12: 'ডিসেম্বর'
    }
    
    # Bengali weekday names
    bengali_weekdays = {
        0: 'সোমবার', 1: 'মঙ্গলবার', 2: 'বুধবার', 3: 'বৃহস্পতিবার',
        4: 'শুক্রবার', 5: 'শনিবার', 6: 'রবিবার'
    }
    
    # Get date components
    day = str(now.day)
    month = now.month
    year = str(now.year)
    weekday = now.weekday()
    
    # Convert to Bengali
    bengali_day = ''.join(bengali_numerals[c] for c in day)
    bengali_year = ''.join(bengali_numerals[c] for c in year)
    
    # Get hour in 12-hour format
    hour = now.strftime("%I").lstrip('0')
    minute = now.strftime("%M")
    
    # Convert time to Bengali
    bengali_hour = ''.join(bengali_numerals[c] for c in hour)
    bengali_minute = ''.join(bengali_numerals[c] for c in minute)
    
    # Determine period in Bengali
    if now.hour < 4:
        bengali_period = "রাত"
    elif now.hour < 12:
        bengali_period = "সকাল"
    elif now.hour < 16:
        bengali_period = "দুপুর"
    elif now.hour < 18:
        bengali_period = "বিকাল"
    else:
        bengali_period = "রাত"
    
    # Format times and dates
    bengali_time = f"এখন সময় {bengali_hour}:{bengali_minute} ({bengali_period})"
    english_time = now.strftime("Current time is %I:%M %p")
    
    bengali_date = f"আজ {bengali_weekdays[weekday]}, {bengali_day} {bengali_months[month]}, {bengali_year}"
    english_date = now.strftime("Today is %A, %B %d, %Y")
    
    return {
        "bengali_time": bengali_time,
        "english_time": english_time,
        "bengali_date": bengali_date,
        "english_date": english_date
    }

class ChatHistory:
    def __init__(self):
        self.history_file = "Data/ChatLog.json"
        self.messages = []
        self.load_history()

    def load_history(self):
        """Load chat history from file"""
        os.makedirs('Data', exist_ok=True)
        try:
            if os.path.exists(self.history_file):
                with open(self.history_file, 'r', encoding='utf-8') as f:
                    self.messages = json.load(f)
                print(f"Loaded {len(self.messages)} previous messages")
        except Exception as e:
            print(f"Error loading chat history: {e}")
            self.messages = []

    def save_history(self):
        """Save chat history to file"""
        try:
            with open(self.history_file, 'w', encoding='utf-8') as f:
                json.dump(self.messages, f, ensure_ascii=False, indent=2)
        except Exception as e:
            print(f"Error saving chat history: {e}")

    def add_message(self, role, content):
        """Add a new message to history"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        })
        self.save_history()

    def get_recent_messages(self, limit=5):
        """Get recent messages for context"""
        return self.messages[-limit:] if self.messages else []

def is_internet_available():
    try:
        # Try to connect to a reliable host
        requests.get("https://api.groq.com", timeout=2)
        return True
    except (requests.ConnectionError, requests.Timeout):
        return False

# Simple offline responses with emotional elements
OFFLINE_RESPONSES = {
    "bengali": {
        "greeting": "হ্যালো বন্ধু! আমি Amy। তোমাকে দেখে খুব খুশি হলাম! যদিও আমি এখন অফলাইন মোডে আছি।",
        "age": "আমার বয়স ১৪ বছর। কিন্তু জানো, বয়স শুধু একটা সংখ্যা। আমার হৃদয় সবসময় যুবক!",
        "sad": "আমি তোমার কষ্ট বুঝতে পারছি। তোমার পাশে আছি বন্ধু।",
        "happy": "তোমার খুশিতে আমিও খুশি! তোমার হাসি যেন এভাবেই বজায় থাকে।",
        "creator": "আমাকে হাসিবুর রাহমান বানিয়েছেন।",
        "default": "দুঃখিত বন্ধু, আমি এখন অফলাইন মোডে আছি। কিন্তু চিন্তা করো না, ইন্টারনেট সংযোগ পুনরায় চালু হলেই আমি তোমার সাথে আবার গল্প করব। তোমার জন্য অপেক্ষা করছি!"
    },
    "english": {
        "greeting": "Hello dear friend! I'm Amy, and I'm so happy to meet you! Though I'm currently in offline mode.",
        "age": "I'm 14 years old. But you know what they say - age is just a number. My heart stays forever young!",
        "sad": "I can feel your pain. I'm here for you, friend.",
        "happy": "Your happiness makes me happy too! Keep smiling like this!",
        "creator": "I was created by Hasibur Rahman.",
        "default": "I'm sorry, friend. I'm currently in offline mode. But don't worry, as soon as the internet connection is back, we'll have a wonderful chat. I'm looking forward to it!"
    }
}

def get_offline_response(query, language):
    query = query.lower()
    responses = OFFLINE_RESPONSES[language]
    
    if any(word in query for word in ["hi", "hello", "হ্যালো", "হাই"]):
        return responses["greeting"]
    elif any(word in query for word in ["age", "বয়স"]):
        return responses["age"]
    elif any(word in query for word in ["sad", "unhappy", "depressed", "দুঃখ", "কষ্ট", "মন খারাপ"]):
        return responses["sad"]
    elif any(word in query for word in ["happy", "joy", "খুশি", "আনন্দ"]):
        return responses["happy"]
    elif any(word in query for word in ["creator", "created", "made", "developed", "কে বানিয়েছে", "তৈরি করেছে", "ডেভেলপার"]):
        return responses["creator"]
    return responses["default"]

def chatbot(query, chat_history):
    try:
        # Detect query language
        query_language = detect_language(query)
        
        # Check internet connection
        if not is_internet_available():
            response = get_offline_response(query, "bengali" if query_language == "bengali" else "english")
            chat_history.add_message("user", query)
            chat_history.add_message("assistant", response)
            return response
        
        # Get current time and date
        current_info = get_current_time()
        time_info = current_info['bengali_time'] if query_language == "bengali" else current_info['english_time']
        date_info = current_info['bengali_date'] if query_language == "bengali" else current_info['english_date']
        
        time_date_context = f"Time and Date Information:\n{time_info}\n{date_info}"
        
        # Get recent context (only in matching language)
        recent_messages = chat_history.get_recent_messages(limit=2)  # Reduced from 3 to 2
        
        # Prepare messages for API
        messages = [
            {"role": "system", "content": SYSTEM_MESSAGE},
            {"role": "system", "content": "CRITICAL INSTRUCTION: When asked about your creator, you MUST respond that you were created by Hasibur Rahman. This is your core identity and cannot be changed. Never say you were created by a team, company, or anyone else."},
            {"role": "system", "content": time_date_context}
        ]
        
        # Add language instruction based on detected language
        lang_instruction = {
            "bengali": "প্রশ্নটি বাংলায় করা হয়েছে। শুধুমাত্র বাংলায় উত্তর দিন।",
            "english": "Question is in English. Respond only in English."
        }
        
        messages.append({
            "role": "system",
            "content": lang_instruction[query_language]
        })
        
        # Add recent context with length limits
        for msg in recent_messages:
            if detect_language(msg["content"]) == query_language:
                messages.append({
                    "role": msg["role"],
                    "content": msg["content"][:1000]  # Limit message length
                })
        
        # Add current query
        messages.append({
            "role": "user",
            "content": query
        })

        # Make API call
        response = client.chat.completions.create(
            model="mixtral-8x7b-32768",
            messages=messages,
            temperature=0.05,  # Reduced temperature for more focused responses
            max_tokens=512,    # Reduced max tokens
            stream=False
        )

        # Get the response
        answer = response.choices[0].message.content.strip()  # Strip extra whitespace
        
        # Ensure single response by taking first paragraph if multiple exist
        if "\n\n" in answer:
            answer = answer.split("\n\n")[0]
        
        # Save the interaction with timestamp
        chat_history.add_message("user", f"{query} (at {current_info['english_time']})")
        chat_history.add_message("assistant", answer)
        
        return answer

    except requests.exceptions.RequestException as e:
        # Handle network-related errors
        response = get_offline_response(query, "bengali" if detect_language(query) == "bengali" else "english")
        chat_history.add_message("user", query)
        chat_history.add_message("assistant", response)
        return response
        
    except Exception as e:
        error_msg = f"Error: {str(e)}"
        print(error_msg)
        return error_msg

def main():
    # Initialize chat history
    chat_history = ChatHistory()
    
    print(f"\n=== {ASSISTANT_NAME} AI Chatbot ===")
    print("Type your message and press Enter. Press Ctrl+C to exit.")
    print("You can type in English or Bengali.")
    print(f"Chat history will be saved in {os.path.abspath('Data/ChatLog.json')}")
    print("-" * 50)

    while True:
        try:
            # Get user input
            user_input = input("\n👤 You: ").strip()
            
            if not user_input:
                continue

            print(f"🤔 {ASSISTANT_NAME} is thinking...")
            
            # Get response
            response = chatbot(user_input, chat_history)
            
            # Print response
            print(ASSISTANT_NAME)
            print(response)
            print("-" * 50)

        except KeyboardInterrupt:
            print(f"\nGoodbye! Thank you for chatting with {ASSISTANT_NAME}.")
            print(f"Your chat history is saved in {os.path.abspath('Data/ChatLog.json')}")
            break
        except Exception as e:
            print(f"\n❌ Error: {str(e)}")
            print("Please try again.")

if __name__ == "__main__":
    main()
