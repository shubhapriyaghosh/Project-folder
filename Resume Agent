import streamlit as st
import fitz
import docx 
import os
import tempfile
from jinja2 import Environment, FileSystemLoader
import pdfkit

# Warning control
import warnings
warnings.filterwarnings('ignore')

from crewai import Agent, Task, Crew, LLM
# from crewai_tools import SerperDevTool 


# Add this after the existing imports
env = Environment(loader=FileSystemLoader("templates"))
WKHTMLTOPDF_PATH = "" # Windows

GROQ_API_KEY = ""
if not GROQ_API_KEY:
    st.error("GROQ_API_KEY not found. Please set it.")
    st.stop()

def extract_text_from_pdf(file_path):
    """Extracts text from a PDF file using PyMuPDF."""
    doc = fitz.open(file_path)
    text = ""
    for page in doc:
        text += page.get_text()
    return text

def extract_text_from_docx(file_path):
    """Extracts text from a DOCX file using python-docx."""
    doc = docx.Document(file_path)
    fullText = []
    for para in doc.paragraphs:
        fullText.append(para.text)
    return "\n".join(fullText)

def extract_text_from_resume(file_path):
    """Determines file type and extracts text."""
    if file_path.endswith(".pdf"):
        return extract_text_from_pdf(file_path)
    elif file_path.endswith(".docx"):
        return extract_text_from_docx(file_path)
    else:
        return "Unsupported file format."

llm = LLM(model="groq/llama3-8b-8192", api_key=GROQ_API_KEY)

# Agent 1: Resume Strategist
resume_feedback_agent = Agent(
    role="Professional Resume Strategist",
    goal="Give feedback on the resume to make it stand out in the job market.Give a score of the resume out of 10.",
    verbose=True,
    llm=llm,
    max_iter=2,
    backstory="With a strategic mind and an eye for detail, you excel at providing feedback on resumes to highlight the most relevant skills and experiences."
)

# Task for Resume Strategist Agent: Align Resume with Job Requirements
resume_feedback_task = Task(
    description=(
        """Give feedback on the resume to make it stand out for recruiters. 
        Review every section, including the summary, work experience, skills, and education. Suggest to add relevant sections if they are missing.  
        Also give an overall score to the resume out of 10. This is the resume: {resume}"""
    ),
    expected_output="The overall score of the resume followed by the feedback in bullet points.",
    agent=resume_feedback_agent
)


# Agent 2: Resume Advisor/Writer
resume_advisor_agent = Agent( # Renamed for clarity
    role="Professional Resume Writer",
    goal="Based on the feedback received from Resume Advisor, make changes to the resume to make it stand out in the job market.",
    verbose=True,
    llm=llm,
    max_iter=2,
    backstory="With a strategic mind and an eye for detail, you excel at refining resumes based on the feedback to highlight the most relevant skills and experiences."
)

# Task for Resume Advisor/Writer Agent: Rewrite Resume
resume_advisor_task = Task(
    description=(
        """Rewrite the resume based on the feedback to make it stand out for recruiters. You can adjust and enhance the resume but don't make up facts. 
        Review and update every section, including the summary, work experience, skills, and education to better reflect the candidate's abilities. This is the resume: {resume}"""
    ),
    expected_output="Resume in markdown format that effectively highlights the candidate's qualifications and experiences",
    context=[resume_feedback_task],
    agent=resume_advisor_agent
)

# search_tool = SerperDevTool() # Commented out

# Agent 3: Researcher (Commented out as in original)
# job_researcher = Agent(
#     role = "Senior Recruitment Consultant",
#     goal = "Find the 5 most relevant, recently posted jobs based on the improved resume recieved from resume advisor and the location preference",
#     tools = [search_tool],
#     verbose = True,
#     backstory = """As a senior recruitment consultant your prowess in finding the most relevant jobs based on the resume and location preference is unmatched. 
#     You can scan the resume efficiently, identify the most suitable job roles and search for the best suited recently posted open job positions at the preffered location."""
# )

# research_task = Task(
#     description = """Find the 5 most relevant recent job postings based on the resume recieved from resume advisor and location preference. This is the preferred location: {location} . 
#     Use the tools to gather relevant content and shortlist the 5 most relevant, recent, job openings""",
#     expected_output=(
#         "A bullet point list of the 5 job openings, with the appropriate links and detailed description about each job, in markdown format" 
#     ),
#     agent=job_researcher
# )

# Crew Setup
crew = Crew(
    agents=[resume_feedback_agent, resume_advisor_agent], # Use renamed agents
    tasks=[resume_feedback_task, resume_advisor_task],
    verbose=True
)

# --- Main Application Logic ---
def process_resume_with_crewai(file_path): # Removed location as it's not used currently
    """
    Extracts text from resume, runs CrewAI, and returns results.
    """
    resume_text = extract_text_from_resume(file_path)
    if resume_text == "Unsupported file format." or not resume_text.strip():
        return "Error: Could not extract text from resume or unsupported file format.", None

    st.info(f"Extracted text (first 200 chars): {resume_text[:200]}...")
    
    try:
        result = crew.kickoff(inputs={"resume": resume_text}) # Removed location from inputs

        # Extract outputs
        # Accessing output directly if available, or raw if not
        feedback = resume_feedback_task.output.raw_output if hasattr(resume_feedback_task.output, 'raw_output') else str(resume_feedback_task.output)
        improved_resume = resume_advisor_task.output.raw_output if hasattr(resume_advisor_task.output, 'raw_output') else str(resume_advisor_task.output)
        
        #job_roles = research_task.output.raw.strip("```markdown").strip("```").strip() # If re-enabled
        
        # Clean markdown backticks (CrewAI sometimes adds them)
        feedback = feedback.strip("```markdown").strip("```").strip()
        improved_resume = improved_resume.strip("```markdown").strip("```").strip()

        return feedback, improved_resume
    except Exception as e:
        st.error(f"An error occurred during CrewAI processing: {e}")
        return f"Error: {e}", None


st.set_page_config(layout="wide")
st.title("📄 Resume Feedback & Enhancement")
st.markdown("*Upload your resume to get AI-powered feedback and an improved version.*")
st.caption("*Expected Runtime: ~1-2 Minutes after submission*")

tab1, tab2 = st.tabs(["🤖 AI Feedback & Enhancement", "✏️ Build Resume from Scratch"])
#with tab1:
# Initialize session state for storing results
if 'feedback' not in st.session_state:
    st.session_state.feedback = None
if 'improved_resume' not in st.session_state:
    st.session_state.improved_resume = None
# if 'job_roles' not in st.session_state: # If job roles are re-enabled
#     st.session_state.job_roles = None

# --- Input Column ---
with st.sidebar:
    st.header("Upload Resume")
    uploaded_file = st.file_uploader("Choose a resume file (PDF or DOCX)", type=["pdf", "docx"])
    # location_input = st.text_input("Preferred Location (Optional)", placeholder="e.g., San Francisco") # If job roles are re-enabled
    
    submit_button = st.button("🚀 Get Feedback & Improved Resume")

# --- Processing and Output ---
if submit_button and uploaded_file:
    # Save uploaded file to a temporary path
    with tempfile.NamedTemporaryFile(delete=False, suffix=os.path.splitext(uploaded_file.name)[1]) as tmp_file:
        tmp_file.write(uploaded_file.getvalue())
        temp_file_path = tmp_file.name
    
    try:
        with st.spinner("Your resume is being analyzed by AI agents... Please wait."):
            # Call the processing function (removed location for now)
            feedback, improved_resume = process_resume_with_crewai(temp_file_path)
            
            st.session_state.feedback = feedback
            st.session_state.improved_resume = improved_resume
            # st.session_state.job_roles = job_roles # If re-enabled

    except Exception as e:
        st.error(f"An error occurred: {e}")
        st.session_state.feedback = f"Error during processing: {e}"
        st.session_state.improved_resume = None
        # st.session_state.job_roles = None
    finally:
        if 'temp_file_path' in locals() and os.path.exists(temp_file_path):
            os.unlink(temp_file_path) # Clean up the temporary file

elif submit_button and not uploaded_file:
    st.warning("Please upload a resume file first.")

# --- Display Results ---
if st.session_state.feedback:
    st.subheader("📝 Resume Feedback")
    if "Error:" in st.session_state.feedback:
        st.error(st.session_state.feedback)
    else:
        st.markdown(st.session_state.feedback)
    st.markdown("---")

if st.session_state.improved_resume:
    st.subheader("✨ Improved Resume")
    st.markdown(st.session_state.improved_resume)
    
    # Option to download the improved resume
    st.download_button(
        label="⬇️ Download Improved Resume (Markdown)",
        data=st.session_state.improved_resume,
        file_name="improved_resume.md",
        mime="text/markdown"
    )
    st.markdown("---")

    # if st.session_state.job_roles: # If job roles are re-enabled
    #     st.subheader("🔍 Relevant Job Roles")
    #     st.markdown(st.session_state.job_roles)
    #     st.markdown("---")

# with tab2:
#     st.header("Build Your Resume from Scratch")
    
#     # Initialize session state for dynamic fields
#     if 'edu_count' not in st.session_state:
#         st.session_state.edu_count = 1
#     if 'exp_count' not in st.session_state:
#         st.session_state.exp_count = 1

#     with st.form("resume_builder"):
#         # Personal Information
#         st.subheader("Personal Information")
#         col1, col2 = st.columns(2)
#         with col1:
#             full_name = st.text_input("Full Name")
#             email = st.text_input("Email")
#         with col2:
#             phone = st.text_input("Phone Number")
#             linkedin = st.text_input("LinkedIn Profile")
        
#         # Education Section
#         st.subheader("Education")
#         educations = []
#         for i in range(st.session_state.edu_count):
#             with st.expander(f"Education {i+1}", expanded=True):
#                 col1, col2 = st.columns(2)
#                 with col1:
#                     institution = st.text_input(f"Institution {i+1}", key=f"inst_{i}")
#                     degree = st.text_input(f"Degree {i+1}", key=f"deg_{i}")
#                 with col2:
#                     field = st.text_input(f"Field of Study {i+1}", key=f"field_{i}")
#                     dates = st.text_input(f"Dates {i+1}", key=f"ed_dates_{i}")
#                 educations.append({
#                     "institution": institution,
#                     "degree": degree,
#                     "field": field,
#                     "dates": dates
#                 })
        
#         # Experience Section
#         st.subheader("Work Experience")
#         experiences = []
#         for i in range(st.session_state.exp_count):
#             with st.expander(f"Experience {i+1}", expanded=True):
#                 col1, col2 = st.columns(2)
#                 with col1:
#                     company = st.text_input(f"Company {i+1}", key=f"comp_{i}")
#                     position = st.text_input(f"Position {i+1}", key=f"pos_{i}")
#                 with col2:
#                     location = st.text_input(f"Location {i+1}", key=f"loc_{i}")
#                     dates = st.text_input(f"Dates {i+1}", key=f"ex_dates_{i}")
#                 description = st.text_area(f"Description {i+1}", key=f"desc_{i}")
#                 experiences.append({
#                     "company": company,
#                     "position": position,
#                     "location": location,
#                     "dates": dates,
#                     "description": description
#                 })

#         # Skills Section
#         st.subheader("Skills")
#         skills = st.text_area("Enter skills (comma-separated)", 
#                             help="e.g., Python, Project Management, Data Analysis")

#         # Form Buttons
#         col1, col2, col3 = st.columns(3)
#         with col1:
#             if st.form_submit_button("➕ Add Education"):
#                 st.session_state.edu_count += 1
#                 st.rerun()
#         with col2:
#             if st.form_submit_button("➕ Add Experience"):
#                 st.session_state.exp_count += 1
#                 st.rerun()
#         with col3:
#             submitted = st.form_submit_button("🚀 Generate Resume")

#     if submitted:
#         # Prepare data for template
#         resume_data = {
#             "personal": {
#                 "name": full_name,
#                 "email": email,
#                 "phone": phone,
#                 "linkedin": linkedin
#             },
#             "educations": educations,
#             "experiences": experiences,
#             "skills": [skill.strip() for skill in skills.split(",") if skill.strip()]
#         }

#         # Render HTML template
#         template = env.get_template("modern_resume.html")
#         html_content = template.render(resume_data)

#         # Generate PDF
#         try:
#             config = pdfkit.configuration(wkhtmltopdf=WKHTMLTOPDF_PATH)
#             pdf = pdfkit.from_string(html_content, False, options={
#                 'page-size': 'A4',
#                 'margin-top': '0.5in',
#                 'margin-right': '0.5in',
#                 'margin-bottom': '0.5in',
#                 'margin-left': '0.5in',
#                 'encoding': 'UTF-8',
#             })
            
#             # Show preview and download
#             st.success("Resume generated successfully!")
            
#             col1, col2 = st.columns(2)
#             with col1:
#                 st.download_button(
#                     label="📄 Download PDF Resume",
#                     data=pdf,
#                     file_name="resume.pdf",
#                     mime="application/pdf"
#                 )
#             with col2:
#                 st.markdown("### Preview")
#                 st.components.html(html_content, height=1000, scrolling=True)
            
#         except Exception as e:
#             st.error(f"PDF generation failed: {str(e)}")


# st.markdown("---")
# st.markdown("Powered by [CrewAI](https://crewai.com/) and [Groq](https://groq.com/) with Streamlit UI.")




