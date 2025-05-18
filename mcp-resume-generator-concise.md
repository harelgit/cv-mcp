# MCP Resume Generator for Claude Desktop

## 1. Overview

The MCP Resume Generator represents a new paradigm in resume creation tools by leveraging the Model Context Protocol (MCP) specifically for Claude Desktop integration. Unlike traditional web applications or APIs, this system enables Claude to function as an intelligent interface between users and specialized resume generation tools.

The key innovation is that users have natural conversations with Claude while the AI dynamically generates interactive React components for each resume creation step. These components appear directly within Claude's interface as artifacts, creating a seamless experience where the LLM guides users through collecting information, refining content, selecting designs, and generating the final document.

This approach combines Claude's conversational capabilities with purpose-built resume tools, allowing for:
- Natural language guidance and personalized suggestions 
- Structured data collection through tailored React components
- Iterative refinement of resume content with AI assistance
- Professional design selection and customization
- Final output as polished HTML/PDF ready for job applications

## 2. MCP Integration

### 2.1 Tool Definition Endpoint

```python
@router.get("/mcp/tools")
async def get_tools():
    """Return tool definitions in Claude's function calling format."""
    return {
        "functions": [
            {
                "name": "start_resume_session",
                "description": "Start a new resume creation session",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "session_type": {
                            "type": "string",
                            "enum": ["content_generation", "design_only", "comprehensive"]
                        },
                        "user_has_existing_resume": { "type": "boolean" }
                    },
                    "required": ["session_type"]
                }
            },
            {
                "name": "submit_dialog_choices",
                "description": "Submit user choices from a dialog and get the next step",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "session_id": { "type": "string" },
                        "dialog_id": { "type": "string" },
                        "user_choices": { "type": "object" }
                    },
                    "required": ["session_id", "dialog_id", "user_choices"]
                }
            },
            {
                "name": "generate_resume",
                "description": "Generate the completed resume HTML and PDF link",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "session_id": { "type": "string" },
                        "title": { "type": "string" }
                    },
                    "required": ["session_id"]
                }
            },
            {
                "name": "export_resume_pdf",
                "description": "Convert the generated resume HTML to a PDF",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "resume_id": { "type": "string" },
                        "paper_size": {
                            "type": "string",
                            "enum": ["A4", "Letter", "Legal"]
                        },
                        "margins": {
                            "type": "string",
                            "enum": ["narrow", "normal", "wide"]
                        }
                    },
                    "required": ["resume_id"]
                }
            }
        ]
    }
```

### 2.2 Tool Execution Endpoint

```python
@router.post("/mcp/execute-tool")
async def execute_tool(call: dict):
    """Execute a tool based on the request from Claude Desktop."""
    tool_name = call.get("name")
    arguments = call.get("arguments", {})
    
    handlers = {
        "start_resume_session": start_resume_session,
        "submit_dialog_choices": submit_dialog_choices,
        "generate_resume": generate_resume,
        "export_resume_pdf": export_resume_pdf
    }
    
    if tool_name not in handlers:
        return {"error": f"Unknown tool: {tool_name}"}
    
    try:
        result = await handlers[tool_name](**arguments)
        return result
    except Exception as e:
        return {"error": str(e)}
```

## 3. Core Implementation

### 3.1 Session Management

```python
async def start_resume_session(session_type: str, user_has_existing_resume: bool = False):
    """Start a new resume creation session."""
    session_id = str(uuid.uuid4())
    
    session = {
        "id": session_id,
        "created_at": datetime.now().isoformat(),
        "session_type": session_type,
        "user_data": {},
        "current_dialog": "personal_info"
    }
    
    await session_store.set(f"session:{session_id}", json.dumps(session))
    dialog = await generate_dialog_component("personal_info", session)
    
    return {
        "session_id": session_id,
        "welcome_message": "Let's create your professional resume together.",
        "next_dialog": dialog
    }
```

### 3.2 Dialog Flow

```python
async def submit_dialog_choices(session_id: str, dialog_id: str, user_choices: dict):
    """Process user choices and advance to next dialog."""
    session = json.loads(await session_store.get(f"session:{session_id}"))
    
    # Update session with choices
    if dialog_id == "personal_info":
        session["user_data"]["personal"] = user_choices
    elif dialog_id == "professional_summary":
        session["user_data"]["summary"] = user_choices
    # Additional dialog types...
    
    next_dialog_id = get_next_dialog(dialog_id, session)
    session["current_dialog"] = next_dialog_id
    await session_store.set(f"session:{session_id}", json.dumps(session))
    
    if next_dialog_id:
        next_dialog = await generate_dialog_component(next_dialog_id, session)
        return {
            "session_id": session_id,
            "next_dialog": next_dialog
        }
    else:
        return {
            "session_id": session_id,
            "completion_message": "All information collected! Ready to generate your resume."
        }
```

### 3.3 Component Generation

```python
async def generate_dialog_component(dialog_type: str, session_data: dict):
    """Generate a React dialog component for Claude."""
    # Load prompt template from file
    with open(f"prompts/{dialog_type}.json", 'r') as f:
        prompt_template = json.load(f)
    
    # Generate component using Claude
    client = anthropic.Client(api_key=os.getenv("CLAUDE_API_KEY"))
    
    response = await client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=2000,
        temperature=0.2,
        system=f"You are a specialized Resume Component Generator. {prompt_template['instructions']}",
        messages=[
            {"role": "user", "content": f"Generate a React component for the {dialog_type} section of a resume. Session data: {json.dumps(session_data)}"}
        ]
    )
    
    component_code = response.content[0].text
    component_code = clean_and_validate_component(component_code)
    
    return {
        "type": "application/vnd.ant.react",
        "content": component_code,
        "dialog_id": dialog_type,
        "instructions": prompt_template.get("user_instructions", f"Fill out this {dialog_type.replace('_', ' ')} form.")
    }
```

### 3.4 Resume Generation and Export

```python
async def generate_resume(session_id: str, title: str = None):
    """Generate a complete resume based on session data."""
    session = json.loads(await session_store.get(f"session:{session_id}"))
    
    # Create default title if none provided
    if not title and "personal" in session.get("user_data", {}):
        name = session["user_data"]["personal"].get("name", "")
        title = f"{name} - Resume"
    elif not title:
        title = "Professional Resume"
    
    resume_id = str(uuid.uuid4())
    template_name = session.get("user_data", {}).get("template", "modern")
    resume_html = await generate_resume_html(session, template_name)
    
    # Store the resume
    resume_data = {
        "id": resume_id, "session_id": session_id, "title": title,
        "html": resume_html, "created_at": datetime.now().isoformat()
    }
    
    await resume_store.set(f"resume:{resume_id}", json.dumps(resume_data))
    preview_html = generate_preview_html(resume_html)
    
    return {
        "resume_id": resume_id,
        "title": title,
        "preview_html": preview_html,
        "download_url": f"/resumes/{resume_id}/view"
    }

async def export_resume_pdf(resume_id: str, paper_size: str = "Letter", margins: str = "normal"):
    """Export a resume as PDF."""
    resume_data = json.loads(await resume_store.get(f"resume:{resume_id}"))
    
    margin_values = {"narrow": "0.5in", "normal": "1in", "wide": "1.5in"}
    margin = margin_values.get(margins, "1in")
    
    pdf_bytes = await generate_pdf(
        html=resume_data["html"],
        paper_size=paper_size,
        margin=margin
    )
    
    pdf_filename = f"{resume_id}.pdf"
    pdf_path = os.path.join(PDF_STORAGE_PATH, pdf_filename)
    
    os.makedirs(os.path.dirname(pdf_path), exist_ok=True)
    with open(pdf_path, "wb") as f:
        f.write(pdf_bytes)
    
    resume_data["pdf_path"] = pdf_path
    resume_data["updated_at"] = datetime.now().isoformat()
    await resume_store.set(f"resume:{resume_id}", json.dumps(resume_data))
    
    return {
        "resume_id": resume_id,
        "pdf_url": f"/resumes/{resume_id}/download-pdf"
    }
```

## 4. Prompt Templates and Flow

### 4.1 Resume Section Flow

```python
def get_next_dialog(current_dialog: str, session: dict) -> str:
    """Determine the next dialog based on current dialog and session state."""
    flows = {
        "comprehensive": [
            "personal_info", "professional_summary", "work_experience",
            "education", "skills", "template_selection", "final_review"
        ],
        "content_generation": [
            "personal_info", "professional_summary", "work_experience", 
            "education", "skills"
        ],
        "design_only": [
            "template_selection", "color_scheme", "layout_customization"
        ]
    }
    
    session_type = session.get("session_type", "comprehensive")
    flow = flows.get(session_type, flows["comprehensive"])
    
    try:
        current_index = flow.index(current_dialog)
        if current_index < len(flow) - 1:
            return flow[current_index + 1]
        else:
            return None  # End of flow
    except ValueError:
        return flow[0]  # Dialog not in flow, default to beginning
```

### 4.2 Example Prompt Templates

```json
// personal_info.json
{
  "instructions": "Create a React component that collects personal information including: name, title, email, phone, location, LinkedIn URL and portfolio website. Use only standard Tailwind utility classes (no square bracket notation). Include validation for email and phone. Make the form accessible and mobile-responsive.",
  "user_instructions": "Please fill out your personal information below. All fields marked with * are required."
}

// professional_summary.json
{
  "instructions": "Create a React component for collecting or generating a professional summary. Include options for: career field selection, years of experience input, key skills listing, and a toggle between user-written or AI-generated summary. Provide a strength meter that evaluates the summary's impact. Use standard Tailwind classes only.",
  "user_instructions": "Create your professional summary - this section gives potential employers a quick overview of your qualifications and career highlights."
}

// work_experience.json
{
  "instructions": "Create a React component for collecting work experience information. Include fields for company name, job title, location, dates, and achievements. Implement a multi-entry system for multiple positions with add/remove controls. Include an 'enhance' button that helps users transform plain responsibilities into achievement-oriented bullets. Use standard Tailwind classes only.",
  "user_instructions": "Add your work experience, starting with your most recent position. Focus on achievements rather than responsibilities to make your resume more impactful."
}
```

## 5. Setup and Security

### 5.1 Claude Desktop Setup

```
# In your README.md:

## Claude Desktop Integration

1. Open Claude Desktop
2. Go to Settings > Developer
3. Add a new custom tool:
   - Tool Discovery URL: http://localhost:8000/mcp/tools
   - Tool Invocation URL: http://localhost:8000/mcp/execute-tool
4. Restart Claude Desktop
5. Ask Claude: "Help me create a professional resume"
```

### 5.2 Security Considerations

```python
@router.get("/resumes/{resume_id}/view")
async def view_resume(resume_id: str, token: str = None):
    """View a resume with token authentication."""
    if not token:
        raise HTTPException(status_code=401, detail="Authentication required")
    
    # Validate token
    is_valid = await validate_resume_access_token(token, resume_id)
    if not is_valid:
        raise HTTPException(status_code=403, detail="Invalid or expired token")
    
    # Retrieve resume
    resume_data = json.loads(await resume_store.get(f"resume:{resume_id}"))
    if not resume_data:
        raise HTTPException(status_code=404, detail="Resume not found")
    
    return HTMLResponse(content=resume_data["html"])

async def generate_resume_access_token(resume_id: str, expiration_seconds: int = 3600):
    """Generate a time-limited access token for a resume."""
    import jwt
    import time
    
    payload = {
        "resume_id": resume_id,
        "exp": int(time.time()) + expiration_seconds,
        "type": "resume_access"
    }
    
    token = jwt.encode(payload, os.getenv("JWT_SECRET"), algorithm="HS256")
    return token
```

## 6. Technology Stack

- **Backend**: Python 3.9+ with FastAPI
- **Storage**: Redis for sessions, PostgreSQL for persistence
- **LLM Integration**: Claude API (Anthropic)
- **Frontend**: React components (dynamically generated)
- **Styling**: Tailwind CSS (standard utility classes)
- **PDF Generation**: WeasyPrint or wkhtmltopdf
- **Deployment**: Docker for containerization

## 7. Implementation Tips

1. **Component Generation Optimization**:
   - Cache commonly used components to reduce LLM API calls
   - Pre-validate prompt templates to ensure consistent outputs
   - Implement retry logic for component generation failures

2. **Error Handling Strategy**:
   - Provide clear error messages through Claude's conversational interface
   - Implement session recovery for interrupted flows
   - Validate user inputs before saving to session store

3. **Security Best Practices**:
   - Encrypt sensitive user information in storage
   - Implement token-based authentication for resume access
   - Set appropriate TTL for sessions and generated PDFs

4. **Testing Approach**:
   - Unit test each tool handler function independently
   - Create integration tests for the complete resume generation flow
   - Test component generation with various session states
   - Validate PDF output across different platforms and browsers
