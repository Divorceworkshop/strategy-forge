import streamlit as st
import json
import os
from datetime import datetime
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail, Email, To, Content

# Configure page
st.set_page_config(
    page_title="Divorce Strategy Advisor",
    page_icon="🧭",
    layout="wide"
)

# Initialize session state
if 'current_step' not in st.session_state:
    st.session_state.current_step = 0
if 'responses' not in st.session_state:
    st.session_state.responses = {}
if 'show_results' not in st.session_state:
    st.session_state.show_results = False
if 'user_id' not in st.session_state:
    st.session_state.user_id = f"user_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
if 'admin_mode' not in st.session_state:
    st.session_state.admin_mode = False
if 'user_email' not in st.session_state:
    st.session_state.user_email = ""

# Question data
QUESTIONS = [
    {
        "id": "approach",
        "question": "What best describes your current approach to your divorce?",
        "options": [
            ("people_please", "I tend to people-please and avoid conflict, even now."),
            ("fair", "I want things to be fair and by the book — a win-win if possible."),
            ("angry", "I'm angry and want to make my ex pay for what they did.")
        ]
    },
    {
        "id": "lawyer",
        "question": "Do you currently have a lawyer or legal representative?",
        "options": [
            ("confident", "Yes, and I feel confident with their support."),
            ("unsure", "Yes, but I'm unsure they're the right fit for me."),
            ("none", "No, I'm managing this on my own for now.")
        ]
    },
    {
        "id": "emotional",
        "question": "How are you doing emotionally these days?",
        "options": [
            ("overwhelmed", "I'm overwhelmed or in survival mode."),
            ("up_down", "I'm up and down but trying to stay grounded."),
            ("stable", "I feel emotionally stable and ready to move forward.")
        ]
    },
    {
        "id": "financial",
        "question": "How do you feel about your financial situation?",
        "options": [
            ("struggling", "I'm struggling or unsure how to manage it on my own."),
            ("anxious", "I'm learning but still have a lot of anxiety."),
            ("confident", "I feel confident managing my finances solo.")
        ]
    },
    {
        "id": "coparenting",
        "question": "Do you have children, and if so, how is co-parenting going?",
        "options": [
            ("no_children", "I don't have children."),
            ("difficult", "Co-parenting is tense or very difficult."),
            ("figuring_out", "It's going okay — we're figuring it out."),
            ("well", "We're doing well with co-parenting.")
        ]
    },
    {
        "id": "overall",
        "question": "Overall, how are you handling your divorce?",
        "options": [
            ("stuck", "It's really hard — I feel stuck or lost."),
            ("coping", "I'm coping, but it's still very emotional."),
            ("recovering", "I feel like I'm recovering and building a new chapter.")
        ]
    }
]

def send_strategy_email(user_email, strategies):
    """Send personalized strategy recommendations via email"""
    sendgrid_key = os.environ.get('SENDGRID_API_KEY')
    
    if not sendgrid_key:
        st.error("Email service is not configured. Please contact support.")
        return False
    
    # Build HTML email content
    html_content = f"""
    <html>
    <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333;">
        <div style="max-width: 600px; margin: 0 auto; padding: 20px;">
            <h1 style="color: #2E86AB; text-align: center;">🧭 Your Personalized Divorce Strategy</h1>
            <p style="text-align: center; color: #666; font-style: italic;">
                A supportive guide to help you navigate this challenging time with clarity and confidence
            </p>
            
            <div style="background: #f8f9fa; padding: 20px; border-radius: 8px; margin: 20px 0;">
                <p><strong>📋 Important Disclaimer:</strong></p>
                <p>While these suggestions are based on sound principles, every divorce situation is unique. 
                It's strongly recommended to contact an experienced divorce strategist to carefully review 
                your specific circumstances and provide personalized guidance tailored to your needs.</p>
            </div>
            
            <p>Based on your responses, here's your personalized action plan. Remember, you don't have to 
            tackle everything at once — choose what feels most urgent or manageable for you right now.</p>
            
            <div style="margin: 30px 0;">
    """
    
    # Add each strategy to the email
    for i, strategy in enumerate(strategies, 1):
        html_content += f"""
                <div style="margin: 30px 0; padding: 20px; border-left: 4px solid #2E86AB; background: #f8f9fa;">
                    <h3 style="color: #2E86AB; margin-top: 0;">{i}. {strategy['title']}</h3>
        """
        
        for item in strategy['content']:
            if item.startswith('**') and item.endswith('**'):
                html_content += f"<p><strong>{item[2:-2]}</strong></p>"
            elif item.startswith('**'):
                html_content += f"<p><strong>{item[2:]}</strong></p>"
            else:
                html_content += f"<p>• {item}</p>"
        
        html_content += "</div>"
    
    # Add additional resources
    html_content += """
                <div style="margin: 30px 0; padding: 20px; background: #e8f4f8; border-radius: 8px;">
                    <h3 style="color: #2E86AB;">📚 Additional Support</h3>
                    <p>Consider these additional resources as you move forward:</p>
                    <ul>
                        <li><strong>Divorce Strategy Coaching:</strong> Contact Karen at The Divorce Workshop 
                            (<a href="mailto:karen@divorceworkshop.ca">karen@divorceworkshop.ca</a>) for personalized guidance</li>
                        <li>Local divorce support groups in your community</li>
                        <li>Professional counseling or therapy focused on life transitions</li>
                        <li>Financial advisors who specialize in divorce planning</li>
                        <li>Meditation or mindfulness apps for daily emotional support</li>
                        <li>Legal aid organizations if you need affordable legal help</li>
                    </ul>
                </div>
                
                <div style="text-align: center; margin: 30px 0; padding: 20px; background: #f0f8ff; border-radius: 8px;">
                    <p style="margin: 0; color: #666;">
                        This personalized strategy was generated based on your responses to our questionnaire.
                        Keep this email for reference as you navigate your divorce journey.
                    </p>
                </div>
            </div>
        </div>
    </body>
    </html>
    """
    
    try:
        sg = SendGridAPIClient(sendgrid_key)
        
        message = Mail(
            from_email=Email("noreply@divorceworkshop.ca", "Divorce Strategy Advisor"),
            to_emails=To(user_email),
            subject="Your Personalized Divorce Strategy Recommendations",
            html_content=Content("text/html", html_content)
        )
        
        response = sg.send(message)
        return True
        
    except Exception as e:
        st.error(f"Could not send email: {str(e)}")
        return False

def save_user_response(user_id, responses, email=""):
    """Save user responses to a JSON file for admin tracking"""
    data_file = "user_responses.json"
    
    # Load existing data or create new
    if os.path.exists(data_file):
        try:
            with open(data_file, 'r') as f:
                all_data = json.load(f)
        except:
            all_data = {}
    else:
        all_data = {}
    
    # Add current user's data
    all_data[user_id] = {
        "email": email,
        "responses": responses,
        "timestamp": datetime.now().isoformat(),
        "completed": True
    }
    
    # Save back to file
    try:
        with open(data_file, 'w') as f:
            json.dump(all_data, f, indent=2)
    except:
        st.error("Could not save user response. Please try again.")

def load_all_responses():
    """Load all user responses for admin view"""
    data_file = "user_responses.json"
    if os.path.exists(data_file):
        try:
            with open(data_file, 'r') as f:
                return json.load(f)
        except:
            return {}
    return {}

def display_admin_panel():
    """Display admin panel with user statistics"""
    st.title("🔐 Admin Dashboard")
    
    all_responses = load_all_responses()
    
    if not all_responses:
        st.info("No user responses recorded yet.")
        return
    
    # Summary statistics
    st.subheader("📊 Usage Statistics")
    col1, col2, col3 = st.columns(3)
    
    with col1:
        st.metric("Total Users", len(all_responses))
    
    with col2:
        completed_users = sum(1 for data in all_responses.values() if data.get('completed', False))
        st.metric("Completed Surveys", completed_users)
    
    with col3:
        if all_responses:
            latest_timestamp = max(data['timestamp'] for data in all_responses.values())
            latest_date = datetime.fromisoformat(latest_timestamp).strftime('%Y-%m-%d')
            st.metric("Latest Response", latest_date)
    
    # User list
    st.subheader("👥 User Responses")
    
    for user_id, data in all_responses.items():
        email = data.get('email', 'No email provided')
        with st.expander(f"User: {user_id} - {data['timestamp'][:10]} - {email}"):
            st.write(f"**Email:** {email}")
            st.write("---")
            
            responses = data['responses']
            
            # Display each response
            for question in QUESTIONS:
                response_key = responses.get(question['id'])
                if response_key:
                    # Find the full response text
                    for option_key, option_text in question['options']:
                        if option_key == response_key:
                            st.write(f"**{question['question']}**")
                            st.write(f"*{option_text}*")
                            st.write("")
                            break
    
    # Export option
    st.subheader("📥 Export Data")
    if st.button("Prepare Data for Download"):
        st.download_button(
            label="Download JSON",
            data=json.dumps(all_responses, indent=2),
            file_name=f"user_responses_{datetime.now().strftime('%Y%m%d')}.json",
            mime="application/json"
        )

def get_strategy_recommendations(responses):
    """Generate personalized strategy recommendations based on user responses"""
    strategies = []
    
    # Analyze responses and build strategy list
    approach = responses.get('approach')
    lawyer = responses.get('lawyer')
    emotional = responses.get('emotional')
    financial = responses.get('financial')
    coparenting = responses.get('coparenting')
    overall = responses.get('overall')
    
    # Emotional Safety and Boundaries
    if approach == 'people_please':
        strategies.append({
            "title": "🛡️ Build Emotional Boundaries",
            "content": [
                "You may be people-pleasing to avoid conflict — this is understandable, but now's the time to build boundaries that protect you.",
                "**Try this:** Practice saying 'I'll get back to you' before responding to any request. It buys you time and builds your decision-making power.",
                "Consider setting specific times for divorce-related conversations to protect your emotional energy.",
                "Remember: Setting boundaries isn't selfish — it's necessary for your wellbeing during this transition."
            ]
        })
    
    if approach == 'angry':
        strategies.append({
            "title": "🧘 Emotional Regulation & Legal Focus",
            "content": [
                "Your anger is valid, but channeling it constructively will serve you better in the long run.",
                "**Try this:** Before making any major decisions, take 24 hours to cool down and consider the long-term consequences.",
                "Work with a therapist who specializes in divorce to process these feelings safely.",
                "Focus your energy on legal strategy rather than emotional retaliation — it will be more effective."
            ]
        })
    
    # Legal Strategy
    if lawyer == 'unsure':
        strategies.append({
            "title": "⚖️ Legal Representation Review",
            "content": [
                "You're unsure about your lawyer — this is a common concern and worth addressing.",
                "**Consider:** Booking a second opinion consultation with another divorce attorney to understand your options.",
                "Prepare specific questions about your case strategy and timeline to evaluate your current representation.",
                "A divorce strategy coach like Karen at The Divorce Workshop (karen@divorceworkshop.ca) can help clarify whether your approach is right for you."
            ]
        })
    
    if lawyer == 'none':
        strategies.append({
            "title": "⚖️ Legal Support Planning",
            "content": [
                "Managing divorce alone can be overwhelming — consider what level of legal support you need.",
                "**Options to explore:** Limited scope representation, legal coaching, or mediation services.",
                "Even a one-time consultation can help you understand your rights and options.",
                "Document everything and keep detailed records of all communications and agreements."
            ]
        })
    
    # Emotional Support
    if emotional in ['overwhelmed', 'up_down']:
        strategies.append({
            "title": "💚 Emotional Support & Stability",
            "content": [
                "What you're feeling is completely normal — divorce is one of life's most stressful experiences.",
                "**Start small:** Choose one daily habit that grounds you, like morning journaling or a 10-minute walk.",
                "Consider joining a divorce support group or working with a therapist who specializes in life transitions.",
                "Remember: Healing isn't linear. Give yourself permission to have difficult days."
            ]
        })
    
    # Financial Guidance
    if financial in ['struggling', 'anxious']:
        strategies.append({
            "title": "💰 Financial Clarity & Planning",
            "content": [
                "Financial anxiety during divorce is incredibly common — you're not alone in feeling this way.",
                "**First step:** Create a simple budget to understand your current income and expenses.",
                "Consider speaking with a financial advisor who specializes in divorce transitions.",
                "Start building an emergency fund, even if it's just $25 per week — every bit helps create security.",
                "Gather all financial documents and create organized files for easy access during proceedings."
            ]
        })
    
    # Co-parenting Support
    if coparenting == 'difficult':
        strategies.append({
            "title": "👨‍👩‍👧‍👦 Co-Parenting Communication",
            "content": [
                "High-conflict co-parenting is exhausting — parallel parenting might help reduce direct contact while keeping kids stable.",
                "**Communication tip:** Use written communication (text/email) to keep interactions factual and documented.",
                "Develop standard responses to common situations: 'I'll consider that and get back to you' can buy you time.",
                "Focus on what you can control: your responses, your household rules, and your relationship with your children.",
                "Consider working with a family therapist or parenting coordinator if conflict is severe."
            ]
        })
    
    if coparenting == 'figuring_out':
        strategies.append({
            "title": "👨‍👩‍👧‍👦 Co-Parenting Improvement",
            "content": [
                "You're doing better than many — building on this foundation will benefit everyone.",
                "**Focus on consistency:** Develop similar routines and rules between households when possible.",
                "Keep communication child-focused and business-like to reduce emotional triggers.",
                "Create a shared calendar for activities, appointments, and schedule changes.",
                "Celebrate small wins — successful co-parenting is a skill that develops over time."
            ]
        })
    
    # Recovery and Growth
    if overall == 'stuck':
        strategies.append({
            "title": "🌟 Recovery Roadmap",
            "content": [
                "Feeling stuck is part of the process — you're in the hardest part right now, and that's okay.",
                "**Start here:** Focus on just one area at a time rather than trying to fix everything at once.",
                "Create a simple daily routine that includes self-care, even if it's just 10 minutes.",
                "Consider working with a divorce strategy coach like Karen at The Divorce Workshop (karen@divorceworkshop.ca) for personalized guidance.",
                "Remember: This phase is temporary. You will move through it, one day at a time."
            ]
        })
    
    if overall == 'recovering':
        strategies.append({
            "title": "🚀 Future Planning & Growth",
            "content": [
                "You're in a great place — this recovery mindset will serve you well moving forward.",
                "**Next steps:** Start envisioning what you want your new life to look like in 6 months and 1 year.",
                "Consider new goals, interests, or changes you've been wanting to make.",
                "Build on the coping strategies that have worked for you so far.",
                "Think about how you can help others going through similar experiences — your insights are valuable."
            ]
        })
    
    # Default strategies for balanced responses
    if not strategies or len(strategies) < 3:
        strategies.append({
            "title": "🧭 General Strategy Foundation",
            "content": [
                "Every divorce journey is unique, and you're navigating yours with thoughtfulness.",
                "**Key principles:** Focus on what you can control, communicate clearly, and prioritize your wellbeing.",
                "Build a support network of professionals and trusted friends/family members.",
                "Document important decisions and communications throughout the process.",
                "Remember that this is a temporary phase leading to a new chapter in your life."
            ]
        })
    
    return strategies

def display_question(question_idx):
    """Display a single question with radio button options"""
    question = QUESTIONS[question_idx]
    
    st.subheader(f"Question {question_idx + 1} of {len(QUESTIONS)}")
    st.write(f"**{question['question']}**")
    
    # Create radio button options
    options = [option[1] for option in question['options']]
    option_keys = [option[0] for option in question['options']]
    
    # Get previous response if it exists
    previous_response = st.session_state.responses.get(question['id'])
    previous_index = None
    if previous_response:
        try:
            previous_index = option_keys.index(previous_response)
        except ValueError:
            previous_index = None
    
    selected = st.radio(
        "Select your response:",
        options,
        index=previous_index,
        key=f"q_{question_idx}"
    )
    
    # Store the response
    if selected:
        selected_index = options.index(selected)
        st.session_state.responses[question['id']] = option_keys[selected_index]
    
    return selected is not None

def display_results():
    """Display the personalized strategy recommendations"""
    st.header("🧭 Your Custom Divorce Strategy")
    
    # Add disclaimer
    st.info("**Important Disclaimer:** While these suggestions are based on sound principles, every divorce situation is unique. It's strongly recommended to contact an experienced divorce strategist to carefully review your specific circumstances and provide personalized guidance tailored to your needs.")
    
    strategies = get_strategy_recommendations(st.session_state.responses)
    
    st.write("Based on your responses, here's your personalized action plan. Remember, you don't have to tackle everything at once — choose what feels most urgent or manageable for you right now.")
    
    for i, strategy in enumerate(strategies, 1):
        st.subheader(f"{i}. {strategy['title']}")
        
        for item in strategy['content']:
            if item.startswith('**') and item.endswith('**'):
                st.write(item)
            elif item.startswith('**'):
                st.write(item)
            else:
                st.write(f"• {item}")
        
        st.write("")  # Add spacing between strategies
    
    # Additional resources section
    st.subheader("📚 Additional Support")
    st.write("Consider these additional resources as you move forward:")
    st.write("• **Divorce Strategy Coaching:** Contact Karen at The Divorce Workshop (karen@divorceworkshop.ca) for personalized guidance")
    st.write("• Local divorce support groups in your community")
    st.write("• Professional counseling or therapy focused on life transitions")
    st.write("• Financial advisors who specialize in divorce planning")
    st.write("• Meditation or mindfulness apps for daily emotional support")
    st.write("• Legal aid organizations if you need affordable legal help")

# Main app logic
def main():
    # Check for admin access
    if st.sidebar.button("Admin Access"):
        st.session_state.admin_mode = not st.session_state.admin_mode
    
    if st.session_state.admin_mode:
        display_admin_panel()
        return
    
    st.title("🧭 Divorce Strategy Advisor")
    st.write("*A supportive guide to help you navigate this challenging time with clarity and confidence*")
    
    if not st.session_state.show_results:
        # Questionnaire phase
        if st.session_state.current_step < len(QUESTIONS):
            # Progress bar
            progress = (st.session_state.current_step + 1) / len(QUESTIONS)
            st.progress(progress)
            
            # Display current question
            question_answered = display_question(st.session_state.current_step)
            
            # Navigation buttons
            col1, col2, col3 = st.columns([1, 2, 1])
            
            with col1:
                if st.session_state.current_step > 0:
                    if st.button("← Previous"):
                        st.session_state.current_step -= 1
                        st.rerun()
            
            with col3:
                if question_answered:
                    if st.session_state.current_step < len(QUESTIONS) - 1:
                        if st.button("Next →"):
                            st.session_state.current_step += 1
                            st.rerun()
                    else:
                        # Email collection before final submission
                        st.write("---")
                        st.subheader("📧 Get Your Results")
                        st.write("Please provide your email address to receive your personalized divorce strategy recommendations.")
                        email = st.text_input("Enter your email address (required):", 
                                            value=st.session_state.user_email,
                                            placeholder="your.email@example.com")
                        
                        if email != st.session_state.user_email:
                            st.session_state.user_email = email
                        
                        if st.button("Send My Strategy →"):
                            if not st.session_state.user_email or '@' not in st.session_state.user_email:
                                st.error("Please enter a valid email address to receive your results.")
                            else:
                                # Generate strategies
                                strategies = get_strategy_recommendations(st.session_state.responses)
                                
                                # Send email with results
                                with st.spinner("Sending your personalized strategy recommendations..."):
                                    if send_strategy_email(st.session_state.user_email, strategies):
                                        # Save user response after successful email send
                                        save_user_response(st.session_state.user_id, st.session_state.responses, st.session_state.user_email)
                                        st.session_state.show_results = True
                                        st.rerun()
                                    else:
                                        st.error("There was an issue sending your email. Please try again or contact support.")
        
        # Show review section if all questions answered
        if len(st.session_state.responses) == len(QUESTIONS):
            st.write("---")
            st.subheader("Review Your Responses")
            
            for i, question in enumerate(QUESTIONS):
                response_key = st.session_state.responses.get(question['id'])
                if response_key:
                    # Find the full response text
                    for option_key, option_text in question['options']:
                        if option_key == response_key:
                            st.write(f"**{question['question']}**")
                            st.write(f"*{option_text}*")
                            st.write("")
                            break
    
    else:
        # Confirmation phase - email sent
        st.success("✅ Your personalized divorce strategy has been sent to your email!")
        
        st.write("---")
        st.subheader("📧 Check Your Email")
        st.write(f"We've sent your personalized divorce strategy recommendations to **{st.session_state.user_email}**")
        
        st.info("""
        **What's in your email:**
        - Personalized strategies based on your responses
        - Actionable advice for your specific situation
        - Karen's contact information for professional coaching
        - Additional resources and support options
        
        **Don't see the email?** Check your spam/junk folder or try again in a few minutes.
        """)
        
        # Option to retake questionnaire
        st.write("---")
        if st.button("🔄 Take Questionnaire Again"):
            st.session_state.current_step = 0
            st.session_state.responses = {}
            st.session_state.show_results = False
            st.session_state.user_email = ""
            st.rerun()

if __name__ == "__main__":
    main()
