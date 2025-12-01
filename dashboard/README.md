# Expert Profiles Dashboard

An interactive dashboard for managing expert profiles related to antitrust litigation strategy, featuring filtering, search, and detailed profile views.

## 🎯 Overview

This dashboard provides structured access to expert profiles across five categories:

| Category | Count | Description |
|----------|-------|-------------|
| ⚖️ Elite Antitrust Attorneys | 6 | Top antitrust litigators with decades of experience |
| 🤖 AI/ML/Robotics Experts | 8 | Leaders from OpenAI, Meta, Google, Boston Dynamics, and more |
| 🔗 Neo4j Graph Database Specialists | 7 | Experts for conspiracy network mapping and relationship analysis |
| 🎓 Elite Law Schools | 5 | Top research centers including Georgetown, Columbia, Harvard |
| 📚 Landmark Legal Cases | 8 | Key precedents establishing antitrust legal framework |

## 📋 Features

### Interactive Filtering
- Filter by category (All, Antitrust, AI/ML, Neo4j, Law Schools, Legal Cases)
- Real-time search across all profiles
- Expandable profile cards with detailed information

### Detailed Profiles Include
- **Attorneys**: Experience, expertise areas, highlights, notable cases, contact info
- **AI/ML Experts**: Research areas, organizations, strategic value for litigation
- **Neo4j Specialists**: Technical expertise, litigation use cases
- **Law Schools**: Clinics, research centers, rankings, strategic partnerships
- **Legal Cases**: Key issues, outcomes, precedent value, strategic relevance

### Strategic Recommendations
- Phase 1: Core team development
- Phase 2: Criminal defense (if applicable)
- Phase 3: Technical analysis & expert witnesses
- Investigation support tools integration

## 🚀 Usage

### Viewing the Dashboard

1. **Local viewing**: Open `index.html` in any modern web browser
2. **Serving locally**: 
   ```bash
   cd dashboard
   python -m http.server 8080
   # Then visit http://localhost:8080
   ```

### Using with Terraform Solution

This dashboard integrates with the main Generative AI Document Summarization solution:

1. Deploy the Terraform infrastructure as documented in the main README
2. Use the document summarization capabilities for litigation document analysis
3. Reference expert profiles for building your litigation team
4. Use Neo4j specialists for conspiracy network visualization

## 📁 Files

| File | Description |
|------|-------------|
| `index.html` | Interactive HTML dashboard with embedded CSS and JavaScript |
| `expert_profiles.json` | Structured JSON data for all expert profiles |
| `README.md` | This documentation file |

## 🔧 Customization

### Adding New Profiles

1. Add entries to `expert_profiles.json` in the appropriate category
2. Update the JavaScript `expertsData` object in `index.html`
3. Profile cards will automatically render with the new data

### Profile Schema

**Attorneys:**
```json
{
  "id": "attorney_XXX",
  "name": "Full Name",
  "firm": "Firm Name",
  "category": "antitrust",
  "experience_years": 25,
  "expertise": ["Area 1", "Area 2"],
  "highlights": ["Highlight 1", "Highlight 2"],
  "strategic_value": "Strategic importance description",
  "contact": {
    "firm_website": "https://...",
    "office_locations": ["City 1", "City 2"]
  }
}
```

**AI/ML Experts:**
```json
{
  "id": "aiml_XXX",
  "name": "Full Name",
  "organization": "Organization Name",
  "category": "ai_ml",
  "expertise": ["Area 1", "Area 2"],
  "highlights": ["Highlight 1", "Highlight 2"],
  "research_areas": ["Area 1", "Area 2"],
  "strategic_value": "Strategic importance description"
}
```

**Neo4j Specialists:**
```json
{
  "id": "neo4j_XXX",
  "name": "Full Name",
  "organization": "Organization Name",
  "category": "neo4j",
  "expertise": ["Area 1", "Area 2"],
  "highlights": ["Highlight 1", "Highlight 2"],
  "use_cases": ["Use Case 1", "Use Case 2"],
  "strategic_value": "Strategic importance description"
}
```

**Law Schools:**
```json
{
  "id": "school_XXX",
  "name": "School Name",
  "category": "law_schools",
  "location": "City, State",
  "rankings": { "us_news_overall": 4 },
  "notable_clinics": ["Clinic 1", "Clinic 2"],
  "research_centers": ["Center 1", "Center 2"],
  "strategic_value": "Strategic importance description"
}
```

**Legal Cases:**
```json
{
  "id": "case_XXX",
  "name": "Case Name",
  "year": "2022",
  "category": "legal_cases",
  "type": "Criminal Antitrust",
  "summary": "Brief case description",
  "key_issues": ["Issue 1", "Issue 2"],
  "outcome": "Case outcome",
  "precedent_value": ["Precedent 1", "Precedent 2"],
  "strategic_relevance": "Strategic importance description"
}
```

## 🤝 Integration with Document Summarization

This dashboard complements the main Terraform GenAI Document Summarization solution by providing:

1. **Expert network for litigation**: Access profiles when building legal teams
2. **Technical expertise mapping**: Identify AI/ML and graph database experts for analysis
3. **Precedent reference**: Quick access to landmark cases and their strategic value
4. **Research partnerships**: Identify law school research centers for collaboration

## 📄 License

Copyright 2024 Google LLC

Licensed under the Apache License, Version 2.0. See [LICENSE](../LICENSE) for details.
