# LinkedIn Insights

A privacy-first, browser-based tool for analyzing your LinkedIn data export and discovering actionable insights about your professional network.

## Features

### Network Health
- **Network Composition**: See which companies and roles dominate your network
- **Connection Growth**: Visualize when you grew your network over time
- **Dormant Connections**: Find people you haven't messaged in 12+ months
- **Strategic Connections**: Discover connections at companies you follow or have saved jobs for

### Career Trajectory
- **Visual Timeline**: See your career journey mapped out
- **Tenure Analysis**: Understand your role duration patterns
- **Career Insights**: Detect transitions and patterns in your career

## Privacy

**Your data never leaves your browser.** All processing happens client-side using JavaScript. No data is sent to any server.

## Usage

1. Download your data from [LinkedIn](https://www.linkedin.com/mypreferences/d/download-my-data)
2. Open `linkedin-insights.html` in your browser
3. Drag and drop your LinkedIn export ZIP file
4. Explore your insights!

## Technical Details

Built with:
- React 18
- Recharts for visualizations
- PapaParse for CSV parsing
- JSZip for ZIP extraction
- Tailwind CSS for styling

All dependencies loaded via CDN - no build step required.
