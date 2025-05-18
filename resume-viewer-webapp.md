# Resume Viewer & PDF Export Web App

## 1. Overview

While the MCP Resume Generator primarily operates through Claude Desktop for conversation and interactive components, a specialized web application component is required to handle resume viewing and PDF export functionality. This document outlines the architecture, endpoints, and implementation details for this crucial part of the system.

## 2. Purpose & Requirements

### 2.1 Key Limitations of Claude Desktop

Claude Desktop provides excellent support for conversational interfaces and React components via artifacts, but it has the following limitations:

- No built-in HTML-to-PDF conversion capabilities
- No way to directly serve downloadable files to users
- Cannot render full HTML resumes with proper styling and layout
- No persistent public URLs for viewing generated content

### 2.2 Core Requirements

The web application component must provide:

1. **Resume Viewing**: Properly formatted HTML display of complete resumes
2. **PDF Generation**: Server-side conversion of HTML resumes to downloadable PDFs
3. **Secure Access Control**: Token-based authentication to protect resume content
4. **Format Options**: Support for different paper sizes, margins, and potentially other formats (DOCX)
5. **Integration**: Seamless connection with the MCP conversation flow

## 3. Architecture

### 3.1 System Integration

```
┌─────────────────┐     ┌───────────────────────┐     ┌─────────────────────┐
│                 │     │                       │     │                     │
│  Claude Desktop │◄───►│   MCP Resume Server   │◄───►│   Session Store     │
│                 │     │   (Python/FastAPI)    │     │     (Redis/DB)      │
│                 │     │                       │     │                     │
└─────────────────┘     └───────────────────────┘     └─────────────────────┘
                                   ▲
                                   │
                                   ▼
                        ┌─────────────────────┐     ┌─────────────────────┐
                        │                     │     │                     │
                        │  Resume Web Viewer  │◄───►│    PDF Generator    │
                        │                     │     │                     │
                        └─────────────────────┘     └─────────────────────┘
```

### 3.2 Deployment Options

The Resume Web Viewer can be implemented in several ways:

1. **Integrated with MCP Server**: As additional endpoints in the same FastAPI application
2. **Separate Microservice**: As a standalone web service that connects to the same data store
3. **Serverless Functions**: As serverless endpoints (AWS Lambda, Vercel Functions, etc.)

Recommendation: Start with option #1 for simplicity, then potentially refactor to option #2 or #3 as usage scales.

## 4. Implementation

### 4.1 Core Endpoints

#### 4.1.1 Resume Viewing Endpoint

```python
@router.get("/resumes/{resume_id}/view")
async def view_resume(resume_id: str, token: str = None):
    """View HTML version of a resume."""
    # Validate access token
    if not await validate_resume_access_token(token, resume_id):
        raise HTTPException(status_code=403, detail="Invalid or expired token")
    
    # Retrieve resume data
    resume_data = json.loads(await resume_store.get(f"resume:{resume_id}"))
    if not resume_data:
        raise HTTPException(status_code=404, detail="Resume not found")
    
    # Return HTML with viewer styling and controls
    html = f"""
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>{resume_data.get('title', 'Resume')}</title>
        <link rel="stylesheet" href="/static/resume-viewer.css">
    </head>
    <body>
        <div class="viewer-controls">
            <a href="/resumes/{resume_id}/download-pdf?token={token}" class="download-btn">
                Download PDF
            </a>
            <div class="format-options">
                <select id="paper-size" onchange="updateDownloadLink()">
                    <option value="Letter">Letter</option>
                    <option value="A4">A4</option>
                    <option value="Legal">Legal</option>
                </select>
                <select id="margins" onchange="updateDownloadLink()">
                    <option value="normal">Normal margins</option>
                    <option value="narrow">Narrow margins</option>
                    <option value="wide">Wide margins</option>
                </select>
            </div>
        </div>
        <div class="resume-container">
            {resume_data['html']}
        </div>
        <script src="/static/resume-viewer.js"></script>
    </body>
    </html>
    """
    
    return HTMLResponse(content=html)
```

#### 4.1.2 PDF Download Endpoint

```python
@router.get("/resumes/{resume_id}/download-pdf")
async def download_resume_pdf(
    resume_id: str, 
    token: str = None, 
    paper_size: str = "Letter", 
    margins: str = "normal"
):
    """Generate and download a PDF version of the resume."""
    # Validate access token
    if not await validate_resume_access_token(token, resume_id):
        raise HTTPException(status_code=403, detail="Invalid or expired token")
    
    # Retrieve resume data
    resume_data = json.loads(await resume_store.get(f"resume:{resume_id}"))
    if not resume_data:
        raise HTTPException(status_code=404, detail="Resume not found")
    
    # Check if we already have a PDF for these parameters
    pdf_key = f"{resume_id}_{paper_size}_{margins}.pdf"
    existing_pdf = await pdf_store.get(pdf_key)
    
    if existing_pdf:
        # Return cached PDF
        return StreamingResponse(
            io.BytesIO(existing_pdf),
            media_type="application/pdf",
            headers={"Content-Disposition": f"attachment; filename=resume.pdf"}
        )
    
    # Set margins based on selection
    margin_values = {
        "narrow": "0.5in",
        "normal": "1in",
        "wide": "1.5in"
    }
    margin = margin_values.get(margins, "1in")
    
    # Generate PDF using WeasyPrint or similar
    pdf_bytes = await generate_pdf(
        html=resume_data["html"],
        paper_size=paper_size,
        margins=margin
    )
    
    # Cache the generated PDF
    await pdf_store.set(pdf_key, pdf_bytes, expire=3600 * 24 * 7)  # Cache for a week
    
    # Return PDF for download
    filename = f"{resume_data.get('title', 'Resume').replace(' ', '_')}.pdf"
    return StreamingResponse(
        io.BytesIO(pdf_bytes),
        media_type="application/pdf",
        headers={"Content-Disposition": f"attachment; filename={filename}"}
    )
```

#### 4.1.3 Optional: Word Document Export

```python
@router.get("/resumes/{resume_id}/download-docx")
async def download_resume_docx(resume_id: str, token: str = None):
    """Generate and download a Word document version of the resume."""
    # Validate access token
    if not await validate_resume_access_token(token, resume_id):
        raise HTTPException(status_code=403, detail="Invalid or expired token")
    
    # Retrieve resume data
    resume_data = json.loads(await resume_store.get(f"resume:{resume_id}"))
    if not resume_data:
        raise HTTPException(status_code=404, detail="Resume not found")
    
    # Check for cached docx
    docx_key = f"{resume_id}.docx"
    existing_docx = await docx_store.get(docx_key)
    
    if existing_docx:
        # Return cached DOCX
        return StreamingResponse(
            io.BytesIO(existing_docx),
            media_type="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            headers={"Content-Disposition": f"attachment; filename=resume.docx"}
        )
    
    # Generate DOCX (using libraries like python-docx or pandoc)
    docx_bytes = await generate_docx(resume_data["html"])
    
    # Cache the generated DOCX
    await docx_store.set(docx_key, docx_bytes, expire=3600 * 24 * 7)  # Cache for a week
    
    # Return DOCX for download
    filename = f"{resume_data.get('title', 'Resume').replace(' ', '_')}.docx"
    return StreamingResponse(
        io.BytesIO(docx_bytes),
        media_type="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        headers={"Content-Disposition": f"attachment; filename={filename}"}
    )
```

### 4.2 Authentication & Security

#### 4.2.1 Token Generation

```python
async def generate_resume_access_token(resume_id: str, expiration_seconds: int = 172800):
    """Generate a secure, time-limited token for resume access (48 hours by default)."""
    import jwt
    import time
    
    # Create a token with expiration
    payload = {
        "resume_id": resume_id,
        "exp": int(time.time()) + expiration_seconds,
        "iat": int(time.time()),
        "type": "resume_access"
    }
    
    # Sign the token
    token = jwt.encode(payload, os.getenv("JWT_SECRET"), algorithm="HS256")
    
    return token
```

#### 4.2.2 Token Validation

```python
async def validate_resume_access_token(token: str, resume_id: str) -> bool:
    """Validate a resume access token."""
    import jwt
    
    try:
        # Decode and verify the token
        payload = jwt.decode(token, os.getenv("JWT_SECRET"), algorithms=["HS256"])
        
        # Check if token is for this resume
        if payload.get("resume_id") != resume_id:
            return False
            
        # Check token type
        if payload.get("type") != "resume_access":
            return False
            
        return True
    except jwt.ExpiredSignatureError:
        # Token has expired
        return False
    except jwt.InvalidTokenError:
        # Invalid token
        return False
```

### 4.3 PDF Generation Implementation

```python
async def generate_pdf(html: str, paper_size: str = "Letter", margins: str = "1in") -> bytes:
    """Generate a PDF from HTML content."""
    # Option 1: Using WeasyPrint
    from weasyprint import HTML, CSS
    from weasyprint.text.fonts import FontConfiguration
    import tempfile
    
    # Create a temporary HTML file with proper doctype and CSS
    with tempfile.NamedTemporaryFile(suffix=".html", mode="w+") as f:
        # Add print-specific CSS
        complete_html = f"""
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <style>
                @page {{ size: {paper_size}; margin: {margins}; }}
                body {{ font-family: 'Helvetica', 'Arial', sans-serif; }}
                /* Additional print-specific CSS */
            </style>
        </head>
        <body>
            {html}
        </body>
        </html>
        """
        
        f.write(complete_html)
        f.flush()
        
        # Create PDF
        font_config = FontConfiguration()
        html = HTML(filename=f.name)
        pdf_bytes = html.write_pdf(font_config=font_config)
    
    return pdf_bytes
    
    # Option 2: Using wkhtmltopdf (alternative implementation)
    # import subprocess
    # import tempfile
    # 
    # with tempfile.NamedTemporaryFile(suffix=".html", mode="w+") as html_file:
    #     html_file.write(html)
    #     html_file.flush()
    #     
    #     with tempfile.NamedTemporaryFile(suffix=".pdf", mode="rb") as pdf_file:
    #         subprocess.call([
    #             'wkhtmltopdf',
    #             '--page-size', paper_size,
    #             '--margin-top', margins,
    #             '--margin-right', margins,
    #             '--margin-bottom', margins,
    #             '--margin-left', margins,
    #             html_file.name,
    #             pdf_file.name
    #         ])
    #         
    #         pdf_file.seek(0)
    #         return pdf_file.read()
```

### 4.4 Frontend Assets

#### 4.4.1 resume-viewer.css

```css
/* Basic styling for the resume viewer */
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin: 0;
  padding: 0;
  background-color: #f5f5f5;
}

.viewer-controls {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  padding: 12px 20px;
  background-color: #fff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  z-index: 1000;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.download-btn {
  display: inline-block;
  padding: 8px 16px;
  background-color: #0066cc;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  font-weight: 500;
  transition: background-color 0.2s;
}

.download-btn:hover {
  background-color: #0055aa;
}

.format-options {
  display: flex;
  gap: 10px;
}

.format-options select {
  padding: 6px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: white;
}

.resume-container {
  max-width: 850px;
  margin: 80px auto 40px;
  padding: 40px;
  background-color: white;
  box-shadow: 0 1px 5px rgba(0, 0, 0, 0.1);
  min-height: 1100px; /* Approximate Letter size paper */
}

/* Responsive adjustments */
@media (max-width: 920px) {
  .resume-container {
    margin: 80px 20px 40px;
    padding: 20px;
  }
}

@media print {
  .viewer-controls {
    display: none;
  }
  
  .resume-container {
    margin: 0;
    padding: 0;
    box-shadow: none;
  }
}
```

#### 4.4.2 resume-viewer.js

```javascript
// Update download link when format options change
function updateDownloadLink() {
  const paperSize = document.getElementById('paper-size').value;
  const margins = document.getElementById('margins').value;
  const downloadBtn = document.querySelector('.download-btn');
  
  // Get current URL and update parameters
  const url = new URL(downloadBtn.href);
  url.searchParams.set('paper_size', paperSize);
  url.searchParams.set('margins', margins);
  
  // Update download button href
  downloadBtn.href = url.toString();
}

// Initialize the page
document.addEventListener('DOMContentLoaded', function() {
  // Initialize download link with default values
  updateDownloadLink();
  
  // Add print button functionality
  const printBtn = document.createElement('button');
  printBtn.textContent = 'Print';
  printBtn.className = 'print-btn';
  printBtn.style.marginLeft = '10px';
  printBtn.style.padding = '8px 16px';
  printBtn.style.backgroundColor = '#555';
  printBtn.style.color = 'white';
  printBtn.style.border = 'none';
  printBtn.style.borderRadius = '4px';
  printBtn.style.cursor = 'pointer';
  
  printBtn.addEventListener('click', function() {
    window.print();
  });
  
  document.querySelector('.viewer-controls').appendChild(printBtn);
});
```

## 5. Integration with MCP Flow

### 5.1 Updated MCP Tool for Resume Generation

```python
async def generate_resume(session_id: str, title: str = None):
    """Generate a complete resume and create access URL with token."""
    # Retrieve session
    session = json.loads(await session_store.get(f"session:{session_id}"))
    
    # Create a default title if none provided
    if not title and "personal" in session.get("user_data", {}):
        name = session["user_data"]["personal"].get("name", "")
        title = f"{name} - Resume"
    elif not title:
        title = "Professional Resume"
    
    # Generate unique resume ID
    resume_id = str(uuid.uuid4())
    
    # Generate HTML using template system
    template_name = session.get("user_data", {}).get("template", "modern")
    resume_html = await generate_resume_html(session, template_name)
    
    # Store the resume
    resume_data = {
        "id": resume_id,
        "session_id": session_id,
        "title": title,
        "html": resume_html,
        "created_at": datetime.now().isoformat()
    }
    
    await resume_store.set(f"resume:{resume_id}", json.dumps(resume_data))
    
    # Generate access token
    token = await generate_resume_access_token(resume_id)
    
    # Create access URL
    base_url = os.getenv("BASE_URL", "http://localhost:8000")
    view_url = f"{base_url}/resumes/{resume_id}/view?token={token}"
    
    # Generate a simplified preview for Claude
    preview_html = generate_preview_html(resume_html)
    
    return {
        "resume_id": resume_id,
        "title": title,
        "preview_html": preview_html,
        "view_url": view_url,
        "token": token
    }
```

### 5.2 Example Conversation Flow

```
User: Great! I think my resume is complete now. Can I see what it looks like?

Claude: Your resume is now complete! I've generated it based on all the information you provided.

[Claude calls generate_resume MCP tool]

Claude: You can view your completed resume using this secure link:

https://resumegenerator.example.com/resumes/res-98765-fghij/view?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

The link will let you see your formatted resume and download it as a PDF. The link is valid for 48 hours.

You can customize the PDF format by selecting different paper sizes and margin settings on the viewer page.

Would you like me to make any additional changes to your resume content before you download it?

User: No, it looks good! Can I also get it in Word format?

Claude: I'd be happy to provide a Word format as well.

[Claude calls export_resume_docx MCP tool]

Claude: You can now download your resume in Word format using this link:

https://resumegenerator.example.com/resumes/res-98765-fghij/download-docx?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

This gives you flexibility to make additional edits directly in Microsoft Word if needed.

Is there anything else you'd like help with for your job application?
```

## 6. Deployment Considerations

### 6.1 Server Requirements

- **Python 3.9+**: For FastAPI and related dependencies
- **PDF Generation Library**: WeasyPrint, wkhtmltopdf, or similar
- **Memory**: Minimum 1GB RAM for PDF generation
- **Storage**: For caching PDFs and storing resume data
- **HTTPS**: Required for secure access to viewer and downloads

### 6.2 Setting Up on Cloud Providers

#### 6.2.1 Vercel Deployment (Serverless)

```json
// vercel.json
{
  "version": 2,
  "builds": [
    {
      "src": "app/main.py",
      "use": "@vercel/python"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "app/main.py"
    }
  ],
  "env": {
    "JWT_SECRET": "@jwt-secret",
    "REDIS_URL": "@redis-url",
    "BASE_URL": "https://resume-generator.vercel.app"
  }
}
```

#### 6.2.2 Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.9-slim

# Install system dependencies for WeasyPrint
RUN apt-get update && apt-get install -y \
    build-essential \
    libcairo2 \
    libpango-1.0-0 \
    libpangocairo-1.0-0 \
    libgdk-pixbuf2.0-0 \
    libffi-dev \
    shared-mime-info \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 6.3 Security Checklist

1. **Secure Tokens**: Use strong JWT secrets and appropriate expiration times
2. **Rate Limiting**: Implement rate limiting on all endpoints
3. **Input Validation**: Validate all input parameters
4. **Content Security Policy**: Implement appropriate CSP headers
5. **HTTPS Only**: Enforce HTTPS for all connections
6. **Cross-Origin Resource Sharing**: Configure CORS appropriately
7. **Protected Static Assets**: Ensure CSS/JS assets cannot be manipulated

## 7. Testing Strategy

### 7.1 Unit Tests

```python
# Example test cases for the PDF generation endpoint
async def test_download_resume_pdf():
    # Test valid token access
    token = await generate_resume_access_token("test-resume-id")
    response = await client.get(f"/resumes/test-resume-id/download-pdf?token={token}")
    assert response.status_code == 200
    assert response.headers["content-type"] == "application/pdf"
    
    # Test invalid token
    response = await client.get("/resumes/test-resume-id/download-pdf?token=invalid-token")
    assert response.status_code == 403
    
    # Test expired token
    expired_token = jwt.encode(
        {
            "resume_id": "test-resume-id",
            "exp": int(time.time()) - 3600,  # Expired 1 hour ago
            "type": "resume_access"
        },
        os.getenv("JWT_SECRET"),
        algorithm="HS256"
    )
    response = await client.get(f"/resumes/test-resume-id/download-pdf?token={expired_token}")
    assert response.status_code == 403
```

### 7.2 Integration Tests

```python
# Example integration test for the full flow
async def test_resume_generation_and_download_flow():
    # Set up test session
    session_id = "test-session-id"
    test_session = {
        "id": session_id,
        "created_at": datetime.now().isoformat(),
        "user_data": {
            "personal": {
                "name": "Test User",
                "email": "test@example.com"
            }
        },
        "template": "modern"
    }
    await session_store.set(f"session:{session_id}", json.dumps(test_session))
    
    # Test resume generation
    generate_response = await generate_resume(session_id)
    assert "resume_id" in generate_response
    assert "view_url" in generate_response
    
    resume_id = generate_response["resume_id"]
    token = generate_response["token"]
    
    # Test resume viewing
    view_response = await client.get(f"/resumes/{resume_id}/view?token={token}")
    assert view_response.status_code == 200
    assert "<!DOCTYPE html>" in view_response.text
    
    # Test PDF download
    pdf_response = await client.get(f"/resumes/{resume_id}/download-pdf?token={token}")
    assert pdf_response.status_code == 200
    assert pdf_response.headers["content-type"] == "application/pdf"
    
    # Test PDF with different options
    pdf_options_response = await client.get(
        f"/resumes/{resume_id}/download-pdf?token={token}&paper_size=A4&margins=narrow"
    )
    assert pdf_options_response.status_code == 200
    assert pdf_options_response.headers["content-type"] == "application/pdf"
```

## 8. Future Enhancements

1. **Additional Export Formats**: Support for more formats beyond PDF and DOCX
2. **Enhanced Viewer Features**: 
   - Print preview mode
   - Theme switching
   - ATS compatibility checking
3. **Resume Analytics**:
   - Tracking of views and downloads
   - Heatmap of resume sections that get the most attention
4. **Social Sharing**:
   - LinkedIn sharing integration
   - Email resume directly to contacts
5. **Resume Version Comparison**:
   - Side-by-side comparison of different versions
   - Tracking changes between versions
