# LLM-Powered React Components for MCP Resume Generator

## 1. Component Generation Architecture

The MCP Resume Generator uses a sophisticated system for dynamically generating interactive React components through LLM prompting. These components appear directly within Claude's interface as artifacts, creating a seamless experience where users can interact with structured input forms while maintaining the natural conversation flow with Claude.

### 1.1 Component Generation Process

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│              │     │              │     │              │
│  Prompt      │────►│  LLM Service │────►│  Component   │
│  Template    │     │  (Claude)    │     │  Validation  │
│              │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
       ▲                                        │
       │                                        ▼
┌──────────────┐                        ┌──────────────┐
│              │                        │              │
│  Session     │                        │  Claude      │
│  Context     │◄───────────────────────┤  Artifact    │
│              │                        │              │
└──────────────┘                        └──────────────┘
```

1. The system selects an appropriate prompt template based on the current dialog type
2. Session context is injected into the prompt to personalize the component
3. The enriched prompt is sent to Claude's API to generate React component code
4. The generated code is validated and cleaned to ensure compatibility
5. The component is formatted as a Claude artifact and delivered to the user

## 2. Prompt Engineering Strategy

### 2.1 Prompt Structure

Each prompt for component generation follows a consistent three-part structure:

```
1. Role & Context Definition
   You are a specialized Resume Component Generator. Your task is to create a React 
   component for [specific resume section] that will be displayed directly in Claude's
   interface as an interactive artifact.

2. Component Requirements
   - Specific fields/inputs required for this section
   - Validation requirements
   - User experience considerations
   - Visual design and layout guidance
   - Special interaction features (e.g., "enhance" buttons for work experience)

3. Technical Constraints
   - Use only standard Tailwind utility classes (no arbitrary values in square brackets)
   - Create a pure functional component that returns JSX
   - Component must be self-contained with appropriate exports
   - All state must be managed within the component
   - Form must handle its own validation and data formatting
```

### 2.2 Tailoring Prompts by Section

Different resume sections require specialized prompts to generate the most effective components:

#### Personal Information
Focus on clear organization of required vs. optional fields, real-time validation of contact information, and a professional layout that establishes the resume's visual identity.

#### Professional Summary
Emphasize the dual-mode capability (write your own or AI-generate), with guidance for optimal length, impact rating, and keyword optimization for the target position.

#### Work Experience
Highlight the multi-entry nature, the transformation of responsibilities into achievement statements, and metrics-focused enhancement suggestions.

#### Skills & Competencies
Focus on categorization, proficiency level visualization, and alignment with industry-specific terminology.

#### Education & Certifications
Emphasize chronological organization, highlighting relevant coursework, and academic achievements.

#### Template Selection
Focus on visual preview capabilities, industry-appropriateness indicators, and customization options.

## 3. Prompt Templates

### 3.1 Base Template Structure

Prompts are stored as JSON files in a dedicated `prompts/` directory:

```json
{
  "instructions": "Detailed instructions for the LLM to generate the component",
  "user_instructions": "Instructions shown to the user in Claude's interface",
  "examples": [
    {
      "description": "Example of strong input for this section",
      "sample": "JSON example of well-formatted data"
    }
  ],
  "constraints": {
    "required_fields": ["field1", "field2"],
    "min_length": {
      "field_name": 50
    },
    "max_length": {
      "field_name": 500
    }
  }
}
```

### 3.2 Example: Personal Information Prompt

```json
{
  "instructions": "Create a React component for collecting personal contact information on a resume. The component should collect the following fields: full name, professional title, email address, phone number, location (city/state/country), LinkedIn URL (optional), and professional website (optional). Implement real-time validation for email (must be valid format) and phone (must be valid format with optional international format). Create a clean, professional form layout with appropriate spacing and visual hierarchy. The form should include a header, clear labels, styled input fields, validation messages, and a 'Copy Choices' button at the bottom that formats all inputs as a structured JSON object and copies it to the clipboard. This Copy Choices functionality is ABSOLUTELY CRITICAL as it's the only way for user selections to be passed back to Claude - the component cannot directly communicate with the chat interface. Include clear instructions for the user explaining they need to paste the copied content back into the chat after clicking the button. Include a progress indicator showing this is the first step in resume creation. Use only standard Tailwind CSS utility classes (no square bracket notation with arbitrary values). Make all components accessible with proper ARIA attributes and keyboard navigation. Ensure the form is fully responsive across all device sizes.",
  
  "user_instructions": "Please fill out your personal information below. Fields marked with * are required. When you're done, click the 'Copy Choices' button and paste the result back into our chat.",
  
  "examples": [
    {
      "description": "Example of a complete and professional personal information section",
      "sample": {
        "name": "Alex Johnson",
        "title": "Senior Software Engineer",
        "email": "alex.johnson@example.com",
        "phone": "555-123-4567",
        "location": "San Francisco, CA",
        "linkedin": "linkedin.com/in/alexjohnson",
        "website": "alexjohnson.dev"
      }
    }
  ],
  
  "constraints": {
    "required_fields": ["name", "email", "phone", "location"],
    "max_length": {
      "name": 60,
      "title": 80,
      "location": 100
    },
    "validation": {
      "email": "^[\\w-\\.]+@([\\w-]+\\.)+[\\w-]{2,4}$",
      "phone": "^[+]?[(]?[0-9]{3}[)]?[-\\s\\.]?[0-9]{3}[-\\s\\.]?[0-9]{4,6}$",
      "website": "^(https?:\\/\\/)?([\\w\\-])+\\.{1}([a-zA-Z]{2,63})([\\/\\w-]*)*\\/?$"
    }
  }
}
```

## 4. Component Generation Implementation

### 4.1 Dynamic Component Generation

```python
async def generate_dialog_component(dialog_type: str, session_data: dict):
    """Generate a React dialog component for Claude."""
    
    # Load prompt template 
    with open(f"prompts/{dialog_type}.json", 'r') as f:
        prompt_template = json.load(f)
    
    # Personalize the prompt based on session context
    personalized_prompt = personalize_prompt(prompt_template, session_data)
    
    # Generate component using Claude
    client = anthropic.Client(api_key=os.getenv("CLAUDE_API_KEY"))
    
    response = await client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=2500,
        temperature=0.2,
        system=f"You are a specialized Resume Component Generator. {personalized_prompt}",
        messages=[
            {"role": "user", "content": f"Generate a React component for the {dialog_type} section of a resume based on the instructions above. The component should be ready to use in Claude's artifact system."}
        ]
    )
    
    component_code = response.content[0].text
    
    # Clean and validate the generated component
    validated_code = clean_and_validate_component(component_code)
    
    # Create the artifact data
    artifact_data = {
        "type": "application/vnd.ant.react",
        "content": validated_code,
        "dialog_id": dialog_type,
        "session_id": session_data["id"],
        "instructions": prompt_template.get("user_instructions", f"Fill out this {dialog_type.replace('_', ' ')} form.")
    }
    
    # Cache the generated component for future use
    cache_key = f"component:{session_data['id']}:{dialog_type}"
    await component_cache.set(cache_key, json.dumps(artifact_data), expire=3600)
    
    return artifact_data

def personalize_prompt(prompt_template: dict, session_data: dict) -> str:
    """Personalize the prompt template based on session data."""
    
    instructions = prompt_template["instructions"]
    
    # Add user name if available
    if "personal" in session_data.get("user_data", {}):
        name = session_data["user_data"]["personal"].get("name")
        if name:
            first_name = name.split()[0]
            instructions = f"The component is for a user named {first_name}. " + instructions
    
    # Add industry context if available
    if "career_field" in session_data.get("user_data", {}):
        field = session_data["user_data"]["career_field"]
        instructions = f"The resume is for the {field} industry. " + instructions
    
    # Add previous choices context for relevant sections
    if "work_experience" in session_data.get("user_data", {}) and len(session_data["user_data"]["work_experience"]) > 0:
        instructions += f"\n\nThe user has already added {len(session_data['user_data']['work_experience'])} work experiences."
    
    # Add progression context
    completed = len(session_data.get("completed_dialogs", []))
    total = len(FLOW_DEFINITIONS.get(session_data.get("session_type", "comprehensive"), []))
    if total > 0:
        progress = int((completed / total) * 100)
        instructions += f"\n\nThis component is part of step {completed+1} of {total} in the resume creation process ({progress}% complete)."
    
    # Add any examples from the template
    if "examples" in prompt_template and len(prompt_template["examples"]) > 0:
        instructions += "\n\nHere are examples of well-formatted data for this section:\n"
        for example in prompt_template["examples"]:
            instructions += f"\n{example['description']}:\n```json\n{json.dumps(example['sample'], indent=2)}\n```"
    
    # Add constraints
    if "constraints" in prompt_template:
        instructions += "\n\nImplement these specific constraints in the component:\n"
        if "required_fields" in prompt_template["constraints"]:
            fields = ", ".join(prompt_template["constraints"]["required_fields"])
            instructions += f"\n- Required fields: {fields}"
        
        if "validation" in prompt_template["constraints"]:
            instructions += "\n- Validation patterns:"
            for field, pattern in prompt_template["constraints"]["validation"].items():
                instructions += f"\n  - {field}: {pattern}"
    
    return instructions
```

### 4.2 Component Validation and Cleaning

```python
def clean_and_validate_component(component_code: str) -> str:
    """Clean and validate the generated React component code."""
    
    # Strip any markdown code block syntax
    code = re.sub(r'```jsx\n', '', component_code)
    code = re.sub(r'```\n?', '', code)
    
    # Check for and remove any unsupported Tailwind classes (arbitrary values)
    code = re.sub(r'(className="[^"]*)\[[^\]]+\]([^"]*")', r'\1\2', code)
    
    # Ensure all state is properly initialized
    if 'useState' in code and not re.search(r'useState\(\s*[\'"][^\'"]*[\'"]\s*\)', code):
        code = re.sub(r'useState\(\s*\)', r'useState("")', code)
    
    # Ensure the component has a proper export default
    if not re.search(r'export\s+default', code):
        component_name_match = re.search(r'(?:function|const)\s+(\w+)', code)
        if component_name_match:
            component_name = component_name_match.group(1)
            code += f"\n\nexport default {component_name};"
        else:
            # If no named component found, wrap the code in a default export
            code = f"const ResumeComponent = () => {{\n  {code}\n}};\n\nexport default ResumeComponent;"
    
    # Validate JSON handling for the "Copy Choices" functionality
    if not re.search(r'JSON\.stringify', code):
        # Add proper JSON stringification if missing
        button_match = re.search(r'<button[^>]*onClick={[^}]*}[^>]*>[^<]*Copy[^<]*</button>', code)
        if button_match:
            button_code = button_match.group(0)
            fixed_button = button_code.replace(
                'onClick={', 
                'onClick={() => {try { navigator.clipboard.writeText(JSON.stringify(formData, null, 2)); alert("Copied to clipboard!"); } catch (e) { console.error(e); alert("Failed to copy"); } }', 
                1
            )
            code = code.replace(button_code, fixed_button)
    
    return code
```

## 5. Component Interaction Patterns

### 5.1 The Critical "Copy Choices" Mechanism

A fundamental limitation of Claude's artifact system is that React components within artifacts **cannot directly communicate back to the chat**. When a user interacts with a component in the artifact panel, their selections and inputs remain isolated within that component and aren't automatically sent back to Claude.

To overcome this limitation, every component must implement a "Copy Choices" button that:

1. Collects all user inputs from the component
2. Formats them as a structured JSON object
3. Copies this JSON to the user's clipboard
4. Instructs the user to paste the result back into the chat

This manual copy-paste workflow is essential because:
- There is no direct API for components to post messages back to Claude
- The artifacts panel is isolated from the main conversation for security reasons
- Form submissions within components cannot automatically update the session state

Example code for implementing the "Copy Choices" functionality:

```jsx
const handleCopyChoices = () => {
  try {
    // Format all form data as a structured JSON object
    const formattedData = JSON.stringify(formData, null, 2);
    
    // Copy to clipboard
    navigator.clipboard.writeText(formattedData);
    
    // Provide visual feedback to user
    setFeedbackMessage("Copied to clipboard! Please paste into the chat.");
    setTimeout(() => setFeedbackMessage(""), 3000);
  } catch (err) {
    console.error("Failed to copy: ", err);
    setFeedbackMessage("Error copying to clipboard. Please try again.");
  }
};

// In the component's return statement:
<div className="mt-6 p-4 bg-blue-50 rounded-md">
  <p className="text-sm text-gray-700 mb-2">
    When you've completed this section, click the button below to copy your
    information, then paste it back into the chat with Claude.
  </p>
  <button
    type="button"
    onClick={handleCopyChoices}
    className="w-full py-2 px-4 bg-blue-600 hover:bg-blue-700 text-white font-medium rounded-md transition duration-200"
  >
    Copy Choices
  </button>
  {feedbackMessage && (
    <p className="text-sm text-center mt-2 text-blue-700">{feedbackMessage}</p>
  )}
</div>
```

Every prompt must explicitly instruct the LLM to include this copy functionality, and Claude must provide clear instructions to users about the copy-paste workflow in each dialog step.

### 5.2 Key Component Interactions

The prompts are engineered to produce components with these key interaction patterns:

#### Data Collection & Validation
- Real-time field validation with visual feedback
- Required field indicators and validation state colors
- Helpful validation messages specific to each field type

#### Enhanced Input Assistance
- "Enhance" buttons for transforming basic content to professional format
- Strength meters for content quality assessment
- Context-specific guidance and examples

#### Multi-Entry Management
- Add/remove controls for sections with multiple entries (work experience, education)
- Drag-and-drop reordering of entries
- Collapsible sections for managing complex content

#### Data Export
- "Copy Choices" functionality that formats all inputs as JSON
- Consistent format across all components for seamless session integration

### 5.2 Example: Work Experience Component Interaction

The work experience component demonstrates sophisticated interaction patterns:

1. **Basic entry form** with company, title, dates, location fields
2. **Responsibilities textarea** for entering basic job duties
3. **"Enhance" button** that transforms each responsibility into achievement-focused bullet
4. **Before/After preview** showing the transformation
5. **Achievement impact rating** visualizing the strength of each bullet
6. **"Add Position" button** for creating multiple job entries
7. **Drag handles** for reordering positions and bullets
8. **"Copy Choices" button** that formats the entire work history as structured JSON

## 6. Prompt Optimization Strategies

### 6.1 Iterative Prompt Refinement

The system uses a feedback loop to continuously improve component generation:

1. **Usage tracking**: Monitor success rates of different prompt variations
2. **Error analysis**: Identify patterns in component generation failures
3. **A/B testing**: Test alternative prompt structures for the same component
4. **Template evolution**: Refine prompts based on performance data

### 6.2 Common Prompt Challenges & Solutions

| Challenge | Solution Approach |
|-----------|------------------|
| Missing "Copy Choices" functionality | Always emphasize in prompts that components must include copy-to-clipboard functionality for all user inputs |
| Inconsistent JSON formatting | Specify exact format for copied data with examples |
| Lack of user guidance | Include explicit instructions in component about copying and pasting |
| Clipboard API failures | Implement error handling and fallback methods for copying |
| Inconsistent styling | Provide explicit Tailwind class examples |
| Missing validation | Include specific validation patterns in constraints |
| Over-complex components | Set explicit complexity boundaries in prompt |
| Import statements | Include cleaning step to remove unsupported imports |
| Arbitrary Tailwind values | Post-process to replace with standard classes |

### 6.3 Performance Optimization

To minimize latency and API costs:

1. **Component caching**: Store generated components for reuse
2. **Prompt templating**: Use a structured template system rather than full prompt regeneration
3. **Progressive enhancement**: Generate simpler components first, then enhance with session context
4. **Parallel generation**: Generate multiple components in parallel for complex sections

## 7. Example Component Outputs

### 7.1 Personal Information Component

```jsx
const PersonalInfoForm = () => {
  const [formData, setFormData] = useState({
    name: "",
    title: "",
    email: "",
    phone: "",
    location: "",
    linkedin: "",
    website: ""
  });
  
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  
  const validateEmail = (email) => {
    const re = /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/;
    return re.test(email);
  };
  
  const validatePhone = (phone) => {
    const re = /^[+]?[(]?[0-9]{3}[)]?[-\s\.]?[0-9]{3}[-\s\.]?[0-9]{4,6}$/;
    return re.test(phone);
  };
  
  const validateUrl = (url) => {
    if (!url) return true;
    const re = /^(https?:\/\/)?([a-z0-9-]+\.)+[a-z]{2,}(\/[^\s]*)?$/i;
    return re.test(url);
  };
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData({
      ...formData,
      [name]: value
    });
    
    setTouched({
      ...touched,
      [name]: true
    });
  };
  
  const validateField = (name, value) => {
    if (name === "name" && !value) {
      return "Name is required";
    }
    if (name === "email") {
      if (!value) return "Email is required";
      if (!validateEmail(value)) return "Please enter a valid email";
    }
    if (name === "phone") {
      if (!value) return "Phone number is required";
      if (!validatePhone(value)) return "Please enter a valid phone number";
    }
    if (name === "location" && !value) {
      return "Location is required";
    }
    if (name === "website" && value && !validateUrl(value)) {
      return "Please enter a valid URL";
    }
    if (name === "linkedin" && value && !validateUrl(value)) {
      return "Please enter a valid LinkedIn URL";
    }
    
    return "";
  };
  
  const handleBlur = (e) => {
    const { name, value } = e.target;
    const error = validateField(name, value);
    
    setErrors({
      ...errors,
      [name]: error
    });
  };
  
  const handleCopyChoices = () => {
    try {
      navigator.clipboard.writeText(JSON.stringify(formData, null, 2));
      alert("Personal information copied to clipboard!");
    } catch (err) {
      console.error("Failed to copy: ", err);
      alert("Failed to copy to clipboard");
    }
  };
  
  return (
    <div className="max-w-2xl mx-auto p-6 bg-white rounded-lg shadow-md">
      <div className="mb-6">
        <h2 className="text-2xl font-bold text-gray-800 mb-2">Personal Information</h2>
        <div className="h-2 w-full bg-gray-200 rounded-full">
          <div className="h-2 bg-blue-600 rounded-full w-1/7"></div>
        </div>
        <p className="text-gray-600 mt-2">Step 1 of 7: Basic details for your resume header</p>
      </div>
      
      <form className="space-y-6">
        <div>
          <label htmlFor="name" className="block text-gray-700 font-medium mb-1">
            Full Name <span className="text-red-500">*</span>
          </label>
          <input
            type="text"
            id="name"
            name="name"
            value={formData.name}
            onChange={handleChange}
            onBlur={handleBlur}
            className={`w-full px-3 py-2 border rounded-md ${
              errors.name && touched.name ? 'border-red-500' : 'border-gray-300'
            } focus:outline-none focus:ring-2 focus:ring-blue-500`}
            placeholder="John Doe"
            required
          />
          {errors.name && touched.name && (
            <p className="text-red-500 text-sm mt-1">{errors.name}</p>
          )}
        </div>
        
        {/* Additional fields omitted for brevity */}
        
        <div className="pt-4">
          <button
            type="button"
            onClick={handleCopyChoices}
            className="w-full py-2 px-4 bg-blue-600 hover:bg-blue-700 text-white font-medium rounded-md transition duration-200"
          >
            Copy Choices
          </button>
        </div>
      </form>
    </div>
  );
};

export default PersonalInfoForm;
```

## 8. Best Practices for Prompt Development

1. **Always emphasize the Copy Choices requirement**: Every prompt must explicitly instruct the LLM to include copy-to-clipboard functionality, as this is the only way for component data to be returned to the conversation.

2. **Include clear user instructions about copying**: Make sure the component contains explicit guidance telling users they need to paste the copied content back into the chat.

3. **Standardize JSON output format**: Define a consistent structure for the copied JSON data to ensure seamless processing in subsequent steps.

4. **Implement robust clipboard handling**: Include error handling for clipboard operations and provide clear feedback when copying succeeds or fails.

5. **Be explicit about Tailwind constraints**: Clearly state that only standard utility classes should be used, with examples of acceptable vs. unacceptable usage.

6. **Provide clear validation requirements**: Include specific regex patterns for validation and explain the expected behavior for validation errors.

7. **Emphasize self-contained components**: Stress that all functionality must be contained within the component with no external dependencies.

8. **Include examples of desired behavior**: Provide concrete examples of component interactions and output formats.

9. **Set explicit boundaries**: Clearly define what the component should and should not do to avoid scope creep.

10. **Maintain a consistent component style**: Use similar interaction patterns across components to provide a unified user experience.

## 10. Content Editing Capabilities

The MCP Resume Generator supports multiple approaches to content editing, balancing structured component-based input with flexible natural language editing.

### 10.1 Component-Based Content Editing

For structured content editing through React components, specialized editing dialogs are created:

```json
// edit_work_experience.json prompt template
{
  "instructions": "Create a React component for editing an existing work experience entry. The component should receive the current work experience data as a prop and pre-populate all fields. Include company name, job title, location, dates, and achievements fields. Allow users to add, edit, and delete achievement bullets. Implement the same validation and enhancement features as the creation component. Most importantly, include the Copy Choices button that formats the edited data as JSON that matches the expected schema.",
  
  "user_instructions": "Edit your work experience information below. When finished, click 'Copy Choices' and paste the result back into our chat.",
  
  "examples": [
    {
      "description": "Example of a pre-populated work experience entry",
      "current_data": {
        "company": "Acme Corporation",
        "title": "Senior Developer",
        "location": "San Francisco, CA",
        "startDate": "2019-03",
        "endDate": "2023-01",
        "achievements": [
          "Increased application performance by 40% through code optimization",
          "Led a team of 5 developers on the customer portal redesign",
          "Implemented CI/CD pipeline reducing deployment time by 60%"
        ]
      }
    }
  ]
}
```

The component generation includes special consideration for editing functionality:

```javascript
// Example code snippet from generated edit component
const EditWorkExperienceForm = ({ existingData }) => {
  // Initialize state with existing data
  const [formData, setFormData] = useState(existingData || {
    company: "",
    title: "",
    location: "",
    startDate: "",
    endDate: "",
    achievements: []
  });
  
  // Include functions for adding/removing/reordering achievements
  const addAchievement = () => {
    setFormData({
      ...formData,
      achievements: [...formData.achievements, ""]
    });
  };
  
  const removeAchievement = (index) => {
    const updatedAchievements = [...formData.achievements];
    updatedAchievements.splice(index, 1);
    setFormData({
      ...formData, 
      achievements: updatedAchievements
    });
  };
  
  // Rest of component implementation...
}
```

### 10.2 Free-Form Editing via Chat

The system also supports natural language editing through direct chat interaction, providing flexibility alongside structured components:

```python
async def handle_free_form_edit(session_id: str, section: str, edit_instructions: str):
    """Process natural language edit instructions for a resume section."""
    # Retrieve session data
    session_data = json.loads(await session_store.get(f"session:{session_id}"))
    
    if not session_data:
        return {"error": "Session not found"}
    
    # Extract current content for the specified section
    current_content = session_data.get("user_data", {}).get(section, {})
    
    if not current_content:
        return {"error": f"No existing content found for {section}"}
    
    # Use LLM to interpret edit instructions and modify content
    client = anthropic.Client(api_key=os.getenv("CLAUDE_API_KEY"))
    
    edit_prompt = f"""
    You are editing the {section} section of a resume.
    
    Current content:
    ```json
    {json.dumps(current_content, indent=2)}
    ```
    
    User's edit instructions:
    "{edit_instructions}"
    
    Apply these edits while maintaining the same JSON structure.
    Return ONLY the edited JSON without any additional explanation.
    """
    
    response = await client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=1500,
        temperature=0.2,
        system="You are a resume editing assistant. Follow the user's instructions precisely while maintaining the required JSON structure.",
        messages=[
            {"role": "user", "content": edit_prompt}
        ]
    )
    
    # Extract the edited content from the response
    try:
        edited_content = json.loads(response.content[0].text)
        
        # Update session with edited content
        session_data["user_data"][section] = edited_content
        await session_store.set(f"session:{session_id}", json.dumps(session_data))
        
        return {
            "session_id": session_id,
            "section": section,
            "status": "updated",
            "edited_content": edited_content
        }
    except json.JSONDecodeError as e:
        return {"error": f"Failed to parse edited content: {str(e)}"}
```

### 10.3 Hybrid Editing Flow

The MCP Resume Generator combines both editing approaches into a cohesive workflow:

1. **Initial Creation**: Use structured components for initial data collection
2. **Section Review**: Present a review component showing the current content
3. **Edit Options**: Provide users with two editing paths:
   - "Edit in Form": Generate a pre-populated React component for structured editing
   - "Edit with Description": Allow natural language instructions for modifications

Example conversation flow for free-form editing:

```
User: I'd like to edit my professional summary. It's too technical and needs to focus more on leadership.

Claude: I'd be happy to help you edit your professional summary. I see two ways we can approach this:

1. I can provide a form where you can directly edit the current content
2. You can describe the changes you want, and I'll update it for you

Which approach would you prefer?

User: Let's use option 2. I want to emphasize my team leadership experience and mention my success with cross-functional collaboration.

Claude: I'll update your professional summary to emphasize leadership and cross-functional collaboration. Here's what I have now:

[Claude calls handle_free_form_edit and displays the result]

Professional Summary:
"Senior Software Engineer with 8 years of experience leading development teams and building scalable web applications. Proven track record of successful cross-functional collaboration, bringing together product, design, and engineering to deliver impactful solutions. Expertise in React, Node.js, AWS, and microservices architecture with a focus on mentoring junior developers and driving engineering excellence."

Does this updated summary look good to you, or would you like to make further edits?
```

### 10.4 Rich Text Editing Components

For longer content sections like the professional summary, the system supports rich text editing capabilities:

```javascript
// Example rich text editor component for professional summary
import React, { useState, useEffect } from 'react';

const RichTextEditor = ({ initialValue = "", onUpdate }) => {
  const [content, setContent] = useState(initialValue);
  const [wordCount, setWordCount] = useState(0);
  const [showToolbar, setShowToolbar] = useState(false);

  useEffect(() => {
    const count = content.split(/\s+/).filter(Boolean).length;
    setWordCount(count);
    if (onUpdate) onUpdate(content);
  }, [content, onUpdate]);

  // Simple text formatting functions
  const applyFormatting = (format) => {
    const selection = window.getSelection();
    if (selection.rangeCount === 0) return;

    const range = selection.getRangeAt(0);
    const selectedText = range.toString();
    
    if (!selectedText) return;
    
    let formattedText = selectedText;
    
    switch (format) {
      case 'bold':
        formattedText = `**${selectedText}**`;
        break;
      case 'italic':
        formattedText = `*${selectedText}*`;
        break;
      case 'bullet':
        formattedText = `\n• ${selectedText}`;
        break;
      default:
        break;
    }
    
    // Create a new DOM Range and replace the selected text
    const newContent = content.substring(0, range.startOffset) + 
                     formattedText + 
                     content.substring(range.endOffset);
    setContent(newContent);
  };

  return (
    <div className="w-full border rounded-md">
      {showToolbar && (
        <div className="flex gap-2 p-2 border-b bg-gray-50">
          <button 
            onClick={() => applyFormatting('bold')}
            className="p-1 rounded hover:bg-gray-200"
            title="Bold"
          >
            <span className="font-bold">B</span>
          </button>
          <button 
            onClick={() => applyFormatting('italic')}
            className="p-1 rounded hover:bg-gray-200"
            title="Italic"
          >
            <span className="italic">I</span>
          </button>
          <button 
            onClick={() => applyFormatting('bullet')}
            className="p-1 rounded hover:bg-gray-200"
            title="Bullet Point"
          >
            • 
          </button>
        </div>
      )}
      <textarea
        className="w-full p-3 min-h-[150px] focus:outline-none focus:ring-2 focus:ring-blue-500"
        value={content}
        onChange={(e) => setContent(e.target.value)}
        onFocus={() => setShowToolbar(true)}
        placeholder="Write your professional summary here..."
      />
      <div className="flex justify-between text-sm p-2 border-t bg-gray-50">
        <span>{wordCount} words</span>
        <span className="text-gray-500">
          {content.length < 50 ? 'Too short' : 
           content.length > 500 ? 'Too long' : 
           'Good length'}
        </span>
      </div>
    </div>
  );
};

// Usage in the professional summary component
const ProfessionalSummaryForm = () => {
  const [summaryText, setSummaryText] = useState("");
  
  // Rest of component implementation...
  
  return (
    <div className="max-w-2xl mx-auto p-6 bg-white rounded-lg shadow-md">
      <h2 className="text-2xl font-bold text-gray-800 mb-4">Professional Summary</h2>
      
      <RichTextEditor
        initialValue={summaryText}
        onUpdate={setSummaryText}
      />
      
      {/* Rest of the component */}
    </div>
  );
};
```

This multi-faceted approach to content editing provides users with flexibility while maintaining the structure needed for consistent resume generation.
