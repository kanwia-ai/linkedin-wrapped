# LinkedIn Wrapped

A privacy-first, browser-based tool for analyzing your LinkedIn data export and discovering actionable insights about your professional network and career.

**Your data never leaves your browser.** All processing happens client-side.

## Features

### Network Health
- **Intent Mode**: Optimize your view for General Networking, Job Search, Building, or Fundraising
- **Forgotten VIPs**: Surface high-value dormant connections worth re-engaging
- **Strategic Connections**: Find connections at companies you follow or have saved jobs for
- **Network Composition**: See which companies and roles dominate your network

### Career Actions
- **Job Search Funnel**: Visualize your saved-to-applied conversion rate
- **Interview Red Flags Radar**: Identify patterns interviewers might ask about (tenure, job hopping, plateaus)
- **Narrative Builder**: Click any role to get interview talking points

### Relationship Intel
- **Conversation Depth Histogram**: See how many shallow vs deep relationships you have
- **Reciprocity Score**: Understand the balance of your conversations
- **Fading Fast**: Identify once-active relationships that have gone cold
- **Intro Path Builder**: Click a target company to see your warm paths in

### Career Trajectory
- **Tenure Analysis**: Understand your role duration patterns vs industry norms
- **Career Insights**: Detect level transitions and patterns in your career

## Usage

1. Download your data from [LinkedIn](https://www.linkedin.com/mypreferences/d/download-my-data)
   - Select "Request a copy of your data"
   - Choose the data you want (messages, connections, positions, etc.)
   - Wait for LinkedIn to email you the download link
2. Open `linkedin-insights.html` in your browser
3. Enter your LinkedIn profile URL (for message analysis)
4. Drag and drop your LinkedIn export ZIP file
5. Explore your insights!

## Technical Details

Built with:
- React 18 (no build step - runs directly in browser)
- Recharts for visualizations
- PapaParse for CSV parsing
- JSZip for ZIP extraction
- Tailwind CSS for styling

All dependencies loaded via CDN.

## Privacy

- All data processing happens in your browser
- No server, no API calls, no data collection
- Your LinkedIn data never leaves your computer

## License

MIT
