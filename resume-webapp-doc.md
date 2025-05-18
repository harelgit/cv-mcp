# Resume Web App - Design Document

## 1. Overview

The Resume Web App provides a streamlined interface for users to view, edit, and download their resumes created with Claude Desktop. This web application complements the MCP Resume Generator by handling the PDF export functionality that Claude Desktop cannot provide directly, while also allowing basic content editing.

## 2. User Interface Design

### 2.1 Navigation Structure

The application features a minimal navigation structure:

- **Top Navigation Bar**:
  - Logo/App Name
  - User profile menu
  
- **Tab Navigation**:
  - Resumes (default)
  - Profile

### 2.2 Resumes Tab

This is the main view where users manage their resumes:

#### 2.2.1 Resume List

- Displays all user resumes in a clean, vertical list
- Each resume entry shows:
  - Title
  - Last modified date
  - View count
- Clicking a resume selects it for viewing

#### 2.2.2 Resume Preview

When a resume is selected:
- Displays formatted resume content
- Shows key metadata (title, last modified date)
- Provides action buttons:
  - Edit Content
  - Download PDF
  - Advanced Editing (redirects to Claude Desktop)

#### 2.2.3 Resume Editing Mode

Basic editing functionality:
- Professional Summary (text area)
- Work Experience section:
  - Company name
  - Job title
  - Dates
  - Achievements (line-by-line editing)
- Skills section (comma-separated input)
- Education section
- Save/Cancel buttons

### 2.3 Profile Tab

Displays basic user information:
- Profile picture
- Name
- Email address
- Account statistics:
  - Account creation date
  - Number of resumes created
  - Total resume views

## 3. Core Functionality

### 3.1 Resume Viewing

- Clean, readable preview of resume content
- Organized by sections (Summary, Experience, Education, Skills)
- Professional formatting that reflects the PDF output

### 3.2 Basic Content Editing

The web app allows editing of text content:

| Content Type | Editing Capability |
|--------------|-------------------|
| Professional Summary | Full text editing |
| Work Experience | Company, title, dates, achievements |
| Skills | Add/remove skills via comma-separated list |
| Education | Institution, degree, dates |

### 3.3 PDF Generation and Download

- One-click PDF generation
- Maintains formatting from selected template
- Applies user-specified paper size and margins
- Shows loading state during generation

### 3.4 Claude Desktop Integration

For more complex changes, the app redirects to Claude Desktop:

- Template changes
- Design customization
- Layout adjustments
- Format restructuring

When users select "Advanced Editing," a modal dialog appears explaining the redirection to Claude Desktop.

## 4. User Experience Flow

### 4.1 Basic Content Editing Flow

1. User selects a resume from the list
2. User clicks "Edit Content" button
3. Edit form appears with current content pre-populated
4. User makes desired text changes
5. User clicks "Save Changes"
6. System displays loading indicator
7. Updated resume appears in preview

### 4.2 Advanced Editing Flow

1. User selects a resume from the list
2. User clicks "Advanced Editing" button
3. System displays modal dialog explaining Claude Desktop redirection
4. User confirms redirection
5. System redirects to Claude Desktop with context about the resume
6. User makes advanced changes in Claude
7. Upon completion, Claude provides updated link to web app

### 4.3 PDF Generation Flow

1. User selects a resume from the list
2. User clicks "Download PDF" button
3. System displays loading indicator
4. PDF is generated server-side
5. Browser initiates download of the PDF file

## 5. Design Elements

### 5.1 Color Palette

- Primary: Blue (#3B82F6)
- Background: Light Gray (#F3F4F6)
- Text: Dark Gray (#1F2937)
- Accents: 
  - Success/Positive: Green (#10B981)
  - Warning/Note: Yellow (#FBBF24)
  - Error: Red (#EF4444)

### 5.2 Typography

- Font Family: System UI stack (native fonts)
- Headings: Semi-bold, 18-24px
- Body Text: Regular, 14-16px
- Captions/Meta: Light, 12-14px

### 5.3 UI Components

- Cards with subtle shadows for content blocks
- Rounded corners (8px radius) on interactive elements
- Consistent padding (16-24px) around content sections
- Input fields with clear focus states

## 6. Technical Implementation

### 6.1 Resume Data Structure

```javascript
{
  id: 'res-98765',
  title: 'Software Engineer Resume',
  lastModified: 'May 18, 2025',
  views: 12,
  sections: {
    summary: 'Senior Software Engineer with 8+ years of experience...',
    experience: [
      {
        company: 'TechCorp Inc.',
        title: 'Senior Software Engineer',
        dates: '2020-Present',
        achievements: [
          'Led development of customer portal serving 50K+ daily users...',
          'Architected microservices infrastructure with Node.js...',
          'Mentored team of 5 junior developers...'
        ]
      },
      // Additional experiences...
    ],
    education: [
      {
        institution: 'University of California, Berkeley',
        degree: 'BS in Computer Science',
        dates: '2013-2017'
      }
    ],
    skills: ['React', 'Node.js', 'TypeScript', 'AWS', 'MongoDB', 'GraphQL']
  }
}
```

### 6.2 API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/resumes` | GET | Retrieve user's resumes |
| `/api/resumes/:id` | GET | Get specific resume data |
| `/api/resumes/:id` | PUT | Update resume content |
| `/api/resumes/:id/pdf` | GET | Generate and download PDF |
| `/api/resumes/:id/views` | POST | Record a resume view |
| `/api/user/profile` | GET | Retrieve user profile data |

### 6.3 Security Considerations

- JWT-based authentication
- Signed URLs for PDF downloads with expiration
- Content validation on all inputs
- CSRF protection on form submissions

## 7. Integration with MCP Resume Generator

### 7.1 Communication Flow

1. Claude Desktop uses MCP tools to generate resume content
2. When a user completes the resume, the MCP server stores the resume data
3. Claude provides a secure link to the web app with the resume ID
4. Web app retrieves the resume data from the server using the ID
5. When the user makes basic edits, the web app updates the storage directly
6. For advanced editing, the web app redirects to Claude with context

### 7.2 Data Synchronization

- Web app gets real-time data from the MCP server
- Edits made in the web app are immediately saved to the shared storage
- When redirecting to Claude Desktop, the current resume state is passed as context
- Claude Desktop uses this context to continue the conversation with the latest data

## 8. Implementation Notes

### 8.1 Performance Considerations

- Cache rendered PDF files for frequently accessed resumes
- Implement progressive loading for resume list
- Use optimistic UI updates during editing to reduce perceived latency

### 8.2 Accessibility Considerations

- Semantic HTML structure for screen reader compatibility
- Keyboard navigation support for all interactive elements
- Sufficient color contrast (WCAG AA compliance)
- Descriptive alt text for any images

### 8.3 Responsive Design

- Mobile-first approach with progressive enhancement
- Collapsible sections on smaller screens
- Touch-friendly input controls
- Adapted layout for various device sizes

## 9. Future Enhancements

### 9.1 Short-term Improvements

- **Version History**: Track changes and allow reverting to previous versions
- **Share Resume**: Generate shareable links with expiration settings
- **ATS Analysis**: Analyze resume compatibility with Applicant Tracking Systems
- **Export Options**: Additional formats beyond PDF (DOCX, TXT, etc.)

### 9.2 Long-term Vision

- **Resume Analytics**: Detailed stats on resume performance
- **Job Application Tracking**: Connect resume to actual job applications
- **Collaborative Editing**: Allow mentors or peers to suggest changes
- **Template Marketplace**: Custom templates from professional designers

## 10. Conclusion

The Resume Web App provides essential viewing, basic editing, and export functionality that complements Claude Desktop's conversational resume creation experience. By maintaining a minimal, focused interface while seamlessly integrating with Claude for advanced editing, the system creates a cohesive experience that leverages the strengths of both platforms.
