# LinkedIn Insights Tool - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a self-contained, browser-based LinkedIn data insights tool that processes uploaded LinkedIn exports and displays actionable Network Health and Career Trajectory insights.

**Architecture:** Single HTML file with embedded React, Recharts for visualization, PapaParse for CSV parsing, and JSZip for ZIP extraction. All processing happens client-side (privacy-first). User uploads their LinkedIn export ZIP or individual CSVs, data is parsed in-browser, and an interactive dashboard renders.

**Tech Stack:** React 18 (CDN), Recharts (CDN), PapaParse (CDN), JSZip (CDN), Tailwind CSS (CDN)

---

## Task 1: Create Base HTML Shell with Dependencies

**Files:**
- Create: `linkedin-insights.html`

**Step 1: Create the HTML file with CDN dependencies and basic structure**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LinkedIn Insights</title>

    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- React -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>

    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

    <!-- Recharts -->
    <script src="https://unpkg.com/recharts@2.12.7/umd/Recharts.js"></script>

    <!-- PapaParse for CSV -->
    <script src="https://unpkg.com/papaparse@5.4.1/papaparse.min.js"></script>

    <!-- JSZip for ZIP extraction -->
    <script src="https://unpkg.com/jszip@3.10.1/dist/jszip.min.js"></script>

    <script>
        tailwind.config = {
            darkMode: 'class',
            theme: {
                extend: {
                    colors: {
                        linkedin: '#0A66C2',
                    }
                }
            }
        }
    </script>

    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; }
        .gradient-bg { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); }
    </style>
</head>
<body class="bg-gray-900 text-white min-h-screen">
    <div id="root"></div>

    <script type="text/babel">
        // App will go here in subsequent tasks
        function App() {
            return (
                <div className="min-h-screen p-8">
                    <h1 className="text-4xl font-bold text-center mb-8">LinkedIn Insights</h1>
                    <p className="text-center text-gray-400">Upload component coming next...</p>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
```

**Step 2: Test the HTML file opens in browser**

Run: `open linkedin-insights.html`
Expected: Browser opens, shows "LinkedIn Insights" title and placeholder text

**Step 3: Commit**

```bash
git init
git add linkedin-insights.html
git commit -m "feat: create base HTML shell with CDN dependencies"
```

---

## Task 2: Build File Upload Component

**Files:**
- Modify: `linkedin-insights.html`

**Step 1: Add the upload component with drag-and-drop**

Replace the `<script type="text/babel">` section with:

```jsx
const { useState, useCallback } = React;

function FileUpload({ onDataLoaded }) {
    const [isDragging, setIsDragging] = useState(false);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const parseCSV = (text) => {
        return new Promise((resolve) => {
            Papa.parse(text, {
                header: true,
                skipEmptyLines: true,
                complete: (results) => resolve(results.data)
            });
        });
    };

    const processZip = async (file) => {
        setLoading(true);
        setError(null);

        try {
            const zip = await JSZip.loadAsync(file);
            const data = {};

            const fileMapping = {
                'Connections.csv': 'connections',
                'messages.csv': 'messages',
                'Positions.csv': 'positions',
                'Company Follows.csv': 'companyFollows',
                'Skills.csv': 'skills',
                'Education.csv': 'education',
                'Jobs/Saved Jobs.csv': 'savedJobs',
                'Jobs/Job Applications.csv': 'jobApplications',
                'Invitations.csv': 'invitations',
            };

            for (const [fileName, key] of Object.entries(fileMapping)) {
                const zipFile = zip.file(fileName);
                if (zipFile) {
                    const text = await zipFile.async('string');
                    data[key] = await parseCSV(text);
                }
            }

            if (!data.connections || data.connections.length === 0) {
                throw new Error('No connections found. Please upload a valid LinkedIn export.');
            }

            onDataLoaded(data);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    };

    const handleDrop = useCallback((e) => {
        e.preventDefault();
        setIsDragging(false);

        const file = e.dataTransfer.files[0];
        if (file && file.name.endsWith('.zip')) {
            processZip(file);
        } else {
            setError('Please upload a ZIP file from LinkedIn export');
        }
    }, []);

    const handleFileSelect = (e) => {
        const file = e.target.files[0];
        if (file) {
            processZip(file);
        }
    };

    return (
        <div className="max-w-2xl mx-auto">
            <div
                onDragOver={(e) => { e.preventDefault(); setIsDragging(true); }}
                onDragLeave={() => setIsDragging(false)}
                onDrop={handleDrop}
                className={`border-2 border-dashed rounded-2xl p-16 text-center transition-all cursor-pointer
                    ${isDragging ? 'border-linkedin bg-linkedin/10' : 'border-gray-600 hover:border-gray-500'}
                    ${loading ? 'opacity-50 pointer-events-none' : ''}`}
            >
                <input
                    type="file"
                    accept=".zip"
                    onChange={handleFileSelect}
                    className="hidden"
                    id="file-input"
                />
                <label htmlFor="file-input" className="cursor-pointer">
                    <div className="text-6xl mb-4">📊</div>
                    <h2 className="text-2xl font-semibold mb-2">
                        {loading ? 'Processing...' : 'Drop your LinkedIn export here'}
                    </h2>
                    <p className="text-gray-400 mb-4">
                        or click to browse for your ZIP file
                    </p>
                    <p className="text-sm text-gray-500">
                        Your data stays in your browser. Nothing is uploaded to any server.
                    </p>
                </label>
            </div>

            {error && (
                <div className="mt-4 p-4 bg-red-900/50 border border-red-500 rounded-lg text-red-200">
                    {error}
                </div>
            )}

            <div className="mt-8 text-center text-gray-500 text-sm">
                <p>Don't have your export? <a href="https://www.linkedin.com/mypreferences/d/download-my-data" target="_blank" className="text-linkedin hover:underline">Download it from LinkedIn</a></p>
            </div>
        </div>
    );
}

function App() {
    const [data, setData] = useState(null);

    if (!data) {
        return (
            <div className="min-h-screen p-8">
                <h1 className="text-4xl font-bold text-center mb-2">LinkedIn Insights</h1>
                <p className="text-center text-gray-400 mb-12">Discover actionable insights from your professional network</p>
                <FileUpload onDataLoaded={setData} />
            </div>
        );
    }

    return (
        <div className="min-h-screen p-8">
            <h1 className="text-4xl font-bold text-center mb-8">LinkedIn Insights</h1>
            <p className="text-center text-green-400">Data loaded! {data.connections?.length} connections found.</p>
            {/* Dashboard will go here */}
        </div>
    );
}

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
```

**Step 2: Test upload with actual LinkedIn ZIP**

Run: `open linkedin-insights.html`
Action: Drag and drop `Basic_LinkedInDataExport_11-29-2025.zip` onto the upload area
Expected: Shows "Data loaded! 2670 connections found." (or similar count)

**Step 3: Commit**

```bash
git add linkedin-insights.html
git commit -m "feat: add file upload with ZIP extraction and CSV parsing"
```

---

## Task 3: Build Data Processing Utilities

**Files:**
- Modify: `linkedin-insights.html`

**Step 1: Add data processing functions after the parseCSV function**

Add these utility functions inside the `<script type="text/babel">` block, before the components:

```jsx
// Data processing utilities
const DataUtils = {
    // Parse various date formats from LinkedIn exports
    parseDate(dateStr) {
        if (!dateStr) return null;

        // Format: "25 Nov 2025" (Connections.csv)
        const dmyMatch = dateStr.match(/(\d{1,2})\s+(\w{3})\s+(\d{4})/);
        if (dmyMatch) {
            return new Date(`${dmyMatch[2]} ${dmyMatch[1]}, ${dmyMatch[3]}`);
        }

        // Format: "2025-11-28 19:31:41 UTC" (messages.csv)
        const isoMatch = dateStr.match(/(\d{4})-(\d{2})-(\d{2})/);
        if (isoMatch) {
            return new Date(dateStr);
        }

        // Format: "Mon Oct 13 15:35:57 UTC 2025" (Company Follows.csv)
        const longMatch = dateStr.match(/\w{3}\s+(\w{3})\s+(\d{1,2})\s+[\d:]+\s+\w+\s+(\d{4})/);
        if (longMatch) {
            return new Date(`${longMatch[1]} ${longMatch[2]}, ${longMatch[3]}`);
        }

        // Format: "10/19/22, 9:45 AM" (Saved Jobs.csv)
        const usMatch = dateStr.match(/(\d{1,2})\/(\d{1,2})\/(\d{2})/);
        if (usMatch) {
            const year = parseInt(usMatch[3]) + 2000;
            return new Date(`${usMatch[1]}/${usMatch[2]}/${year}`);
        }

        return new Date(dateStr);
    },

    // Get unique message partners and last message date
    getMessagePartners(messages, userProfileUrl) {
        const partners = {};

        messages.forEach(msg => {
            const partnerUrl = msg['SENDER PROFILE URL'] === userProfileUrl
                ? msg['RECIPIENT PROFILE URLS']
                : msg['SENDER PROFILE URL'];
            const partnerName = msg['SENDER PROFILE URL'] === userProfileUrl
                ? msg['TO']
                : msg['FROM'];

            if (!partnerUrl || partnerUrl === userProfileUrl) return;

            const date = this.parseDate(msg['DATE']);
            if (!partners[partnerUrl] || date > partners[partnerUrl].lastMessage) {
                partners[partnerUrl] = {
                    name: partnerName,
                    url: partnerUrl,
                    lastMessage: date,
                    messageCount: (partners[partnerUrl]?.messageCount || 0) + 1
                };
            } else {
                partners[partnerUrl].messageCount++;
            }
        });

        return partners;
    },

    // Analyze network composition by company
    getNetworkComposition(connections) {
        const companies = {};
        const positions = {};

        connections.forEach(conn => {
            const company = conn['Company'] || 'Unknown';
            const position = conn['Position'] || 'Unknown';

            companies[company] = (companies[company] || 0) + 1;
            positions[position] = (positions[position] || 0) + 1;
        });

        return {
            topCompanies: Object.entries(companies)
                .sort((a, b) => b[1] - a[1])
                .slice(0, 15)
                .map(([name, count]) => ({ name, count })),
            topPositions: Object.entries(positions)
                .sort((a, b) => b[1] - a[1])
                .slice(0, 10)
                .map(([name, count]) => ({ name, count }))
        };
    },

    // Get connection growth over time
    getConnectionGrowth(connections) {
        const monthly = {};

        connections.forEach(conn => {
            const date = this.parseDate(conn['Connected On']);
            if (!date || isNaN(date.getTime())) return;

            const key = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}`;
            monthly[key] = (monthly[key] || 0) + 1;
        });

        return Object.entries(monthly)
            .sort((a, b) => a[0].localeCompare(b[0]))
            .map(([month, count]) => ({ month, count }));
    },

    // Find dormant connections (time-based)
    getDormantConnections(connections, messagePartners, monthsThreshold = 12) {
        const now = new Date();
        const thresholdDate = new Date(now.setMonth(now.getMonth() - monthsThreshold));

        return connections
            .map(conn => {
                const connectedDate = this.parseDate(conn['Connected On']);
                const url = conn['URL'];
                const partner = messagePartners[url];

                const lastContact = partner?.lastMessage || connectedDate;
                const monthsSinceContact = lastContact
                    ? Math.floor((new Date() - lastContact) / (1000 * 60 * 60 * 24 * 30))
                    : null;

                return {
                    name: `${conn['First Name']} ${conn['Last Name']}`,
                    company: conn['Company'] || 'Unknown',
                    position: conn['Position'] || 'Unknown',
                    url: url,
                    connectedOn: connectedDate,
                    lastMessage: partner?.lastMessage || null,
                    messageCount: partner?.messageCount || 0,
                    monthsSinceContact,
                    isDormant: !lastContact || lastContact < thresholdDate
                };
            })
            .filter(c => c.isDormant)
            .sort((a, b) => (b.connectedOn || 0) - (a.connectedOn || 0));
    },

    // Find strategic connections (at companies you follow or saved jobs for)
    getStrategicConnections(connections, companyFollows, savedJobs, messagePartners) {
        const targetCompanies = new Set();

        // Add followed companies
        companyFollows?.forEach(f => {
            if (f['Organization']) targetCompanies.add(f['Organization'].toLowerCase());
        });

        // Add companies from saved jobs
        savedJobs?.forEach(j => {
            if (j['Company Name']) targetCompanies.add(j['Company Name'].toLowerCase());
        });

        return connections
            .filter(conn => {
                const company = (conn['Company'] || '').toLowerCase();
                return targetCompanies.has(company);
            })
            .map(conn => {
                const url = conn['URL'];
                const partner = messagePartners[url];

                return {
                    name: `${conn['First Name']} ${conn['Last Name']}`,
                    company: conn['Company'],
                    position: conn['Position'] || 'Unknown',
                    url: url,
                    lastMessage: partner?.lastMessage || null,
                    messageCount: partner?.messageCount || 0,
                    isWarm: partner && partner.messageCount > 0
                };
            })
            .sort((a, b) => (b.messageCount || 0) - (a.messageCount || 0));
    },

    // Analyze career trajectory from positions
    getCareerTrajectory(positions) {
        return positions
            .filter(p => p['Company Name'] && p['Started On'])
            .map(p => {
                const startParts = p['Started On'].split(' ');
                const endParts = p['Finished On'] ? p['Finished On'].split(' ') : null;

                const monthMap = { Jan: 0, Feb: 1, Mar: 2, Apr: 3, May: 4, Jun: 5,
                                   Jul: 6, Aug: 7, Sep: 8, Oct: 9, Nov: 10, Dec: 11 };

                const startDate = new Date(parseInt(startParts[1]) || 2020, monthMap[startParts[0]] || 0);
                const endDate = endParts
                    ? new Date(parseInt(endParts[1]) || 2024, monthMap[endParts[0]] || 11)
                    : new Date();

                const months = Math.max(1, Math.round((endDate - startDate) / (1000 * 60 * 60 * 24 * 30)));

                return {
                    company: p['Company Name'],
                    title: p['Title'],
                    description: p['Description'],
                    location: p['Location'],
                    startDate,
                    endDate: endParts ? endDate : null,
                    isCurrent: !endParts,
                    tenureMonths: months
                };
            })
            .sort((a, b) => b.startDate - a.startDate);
    }
};
```

**Step 2: Verify no syntax errors**

Run: `open linkedin-insights.html`
Expected: Page loads without JavaScript errors in console

**Step 3: Commit**

```bash
git add linkedin-insights.html
git commit -m "feat: add data processing utilities for insights"
```

---

## Task 4: Build Network Health Dashboard Section

**Files:**
- Modify: `linkedin-insights.html`

**Step 1: Add the Network Health components**

Add these components after DataUtils, before App:

```jsx
const { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, LineChart, Line, PieChart, Pie, Cell } = Recharts;

const COLORS = ['#0A66C2', '#00A0DC', '#86D4F5', '#0073B1', '#004182', '#7FC15E', '#E68523', '#DD5143'];

function StatCard({ label, value, subtitle }) {
    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <p className="text-gray-400 text-sm mb-1">{label}</p>
            <p className="text-3xl font-bold text-white">{value}</p>
            {subtitle && <p className="text-gray-500 text-sm mt-1">{subtitle}</p>}
        </div>
    );
}

function NetworkComposition({ data }) {
    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <h3 className="text-xl font-semibold mb-4">Top Companies in Your Network</h3>
            <ResponsiveContainer width="100%" height={300}>
                <BarChart data={data.slice(0, 10)} layout="vertical">
                    <XAxis type="number" stroke="#666" />
                    <YAxis type="category" dataKey="name" stroke="#666" width={150} tick={{ fontSize: 12 }} />
                    <Tooltip
                        contentStyle={{ backgroundColor: '#1f2937', border: 'none', borderRadius: '8px' }}
                        labelStyle={{ color: '#fff' }}
                    />
                    <Bar dataKey="count" fill="#0A66C2" radius={[0, 4, 4, 0]} />
                </BarChart>
            </ResponsiveContainer>
        </div>
    );
}

function ConnectionGrowth({ data }) {
    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <h3 className="text-xl font-semibold mb-4">Connection Growth Over Time</h3>
            <ResponsiveContainer width="100%" height={250}>
                <LineChart data={data}>
                    <XAxis dataKey="month" stroke="#666" tick={{ fontSize: 10 }} />
                    <YAxis stroke="#666" />
                    <Tooltip
                        contentStyle={{ backgroundColor: '#1f2937', border: 'none', borderRadius: '8px' }}
                        labelStyle={{ color: '#fff' }}
                    />
                    <Line type="monotone" dataKey="count" stroke="#0A66C2" strokeWidth={2} dot={false} />
                </LineChart>
            </ResponsiveContainer>
        </div>
    );
}

function DormantConnections({ connections }) {
    const [showAll, setShowAll] = useState(false);
    const displayed = showAll ? connections : connections.slice(0, 10);

    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <div className="flex justify-between items-center mb-4">
                <div>
                    <h3 className="text-xl font-semibold">Dormant Connections</h3>
                    <p className="text-gray-400 text-sm">People you haven't messaged in 12+ months</p>
                </div>
                <span className="bg-yellow-600/30 text-yellow-400 px-3 py-1 rounded-full text-sm font-medium">
                    {connections.length} connections
                </span>
            </div>

            <div className="space-y-3 max-h-96 overflow-y-auto">
                {displayed.map((conn, i) => (
                    <a
                        key={i}
                        href={conn.url}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="flex justify-between items-center p-3 bg-gray-700/50 rounded-lg hover:bg-gray-700 transition"
                    >
                        <div>
                            <p className="font-medium">{conn.name}</p>
                            <p className="text-sm text-gray-400">{conn.position} at {conn.company}</p>
                        </div>
                        <div className="text-right text-sm">
                            <p className="text-gray-400">
                                {conn.lastMessage
                                    ? `Last msg: ${conn.lastMessage.toLocaleDateString()}`
                                    : 'Never messaged'}
                            </p>
                            <p className="text-gray-500">
                                Connected: {conn.connectedOn?.toLocaleDateString() || 'Unknown'}
                            </p>
                        </div>
                    </a>
                ))}
            </div>

            {connections.length > 10 && (
                <button
                    onClick={() => setShowAll(!showAll)}
                    className="mt-4 w-full py-2 text-linkedin hover:underline"
                >
                    {showAll ? 'Show less' : `Show all ${connections.length} dormant connections`}
                </button>
            )}
        </div>
    );
}

function StrategicConnections({ connections }) {
    const [showAll, setShowAll] = useState(false);
    const warmConnections = connections.filter(c => c.isWarm);
    const coldConnections = connections.filter(c => !c.isWarm);
    const displayed = showAll ? connections : connections.slice(0, 10);

    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <div className="flex justify-between items-center mb-4">
                <div>
                    <h3 className="text-xl font-semibold">Strategic Connections</h3>
                    <p className="text-gray-400 text-sm">People at companies you follow or saved jobs for</p>
                </div>
                <div className="flex gap-2">
                    <span className="bg-green-600/30 text-green-400 px-3 py-1 rounded-full text-sm font-medium">
                        {warmConnections.length} warm
                    </span>
                    <span className="bg-blue-600/30 text-blue-400 px-3 py-1 rounded-full text-sm font-medium">
                        {coldConnections.length} cold
                    </span>
                </div>
            </div>

            <div className="space-y-3 max-h-96 overflow-y-auto">
                {displayed.map((conn, i) => (
                    <a
                        key={i}
                        href={conn.url}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="flex justify-between items-center p-3 bg-gray-700/50 rounded-lg hover:bg-gray-700 transition"
                    >
                        <div className="flex items-center gap-3">
                            <span className={`w-2 h-2 rounded-full ${conn.isWarm ? 'bg-green-400' : 'bg-blue-400'}`} />
                            <div>
                                <p className="font-medium">{conn.name}</p>
                                <p className="text-sm text-gray-400">{conn.position} at {conn.company}</p>
                            </div>
                        </div>
                        <div className="text-right text-sm text-gray-400">
                            {conn.messageCount > 0
                                ? `${conn.messageCount} messages`
                                : 'No messages yet'}
                        </div>
                    </a>
                ))}
            </div>

            {connections.length > 10 && (
                <button
                    onClick={() => setShowAll(!showAll)}
                    className="mt-4 w-full py-2 text-linkedin hover:underline"
                >
                    {showAll ? 'Show less' : `Show all ${connections.length} strategic connections`}
                </button>
            )}
        </div>
    );
}

function NetworkHealth({ data, insights }) {
    return (
        <div className="space-y-6">
            <h2 className="text-2xl font-bold flex items-center gap-2">
                <span>🔗</span> Network Health
            </h2>

            <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
                <StatCard
                    label="Total Connections"
                    value={data.connections?.length?.toLocaleString() || 0}
                />
                <StatCard
                    label="Companies in Network"
                    value={insights.composition.topCompanies.length}
                    subtitle="unique companies"
                />
                <StatCard
                    label="Dormant Connections"
                    value={insights.dormant.length}
                    subtitle="12+ months silent"
                />
                <StatCard
                    label="Strategic Connections"
                    value={insights.strategic.length}
                    subtitle="at target companies"
                />
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                <NetworkComposition data={insights.composition.topCompanies} />
                <ConnectionGrowth data={insights.growth} />
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                <DormantConnections connections={insights.dormant} />
                <StrategicConnections connections={insights.strategic} />
            </div>
        </div>
    );
}
```

**Step 2: Test renders without errors**

Run: `open linkedin-insights.html`
Expected: No console errors (components defined but not yet used)

**Step 3: Commit**

```bash
git add linkedin-insights.html
git commit -m "feat: add Network Health dashboard components"
```

---

## Task 5: Build Career Trajectory Dashboard Section

**Files:**
- Modify: `linkedin-insights.html`

**Step 1: Add Career Trajectory components after NetworkHealth**

```jsx
function CareerTimeline({ positions }) {
    const totalYears = positions.length > 0
        ? Math.ceil((new Date() - positions[positions.length - 1]?.startDate) / (1000 * 60 * 60 * 24 * 365))
        : 0;

    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <h3 className="text-xl font-semibold mb-6">Career Timeline</h3>

            <div className="relative">
                {/* Timeline line */}
                <div className="absolute left-4 top-0 bottom-0 w-0.5 bg-gray-600" />

                <div className="space-y-6">
                    {positions.map((pos, i) => (
                        <div key={i} className="relative pl-12">
                            {/* Timeline dot */}
                            <div className={`absolute left-2.5 w-3 h-3 rounded-full ${pos.isCurrent ? 'bg-green-400' : 'bg-linkedin'}`} />

                            <div className="bg-gray-700/50 rounded-lg p-4">
                                <div className="flex justify-between items-start mb-2">
                                    <div>
                                        <h4 className="font-semibold text-lg">{pos.title}</h4>
                                        <p className="text-linkedin">{pos.company}</p>
                                    </div>
                                    <div className="text-right text-sm">
                                        <p className="text-gray-300">
                                            {pos.startDate?.toLocaleDateString('en-US', { month: 'short', year: 'numeric' })}
                                            {' → '}
                                            {pos.isCurrent ? 'Present' : pos.endDate?.toLocaleDateString('en-US', { month: 'short', year: 'numeric' })}
                                        </p>
                                        <p className="text-gray-500">
                                            {Math.floor(pos.tenureMonths / 12)}y {pos.tenureMonths % 12}m
                                        </p>
                                    </div>
                                </div>
                                {pos.location && (
                                    <p className="text-sm text-gray-400">📍 {pos.location}</p>
                                )}
                            </div>
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
}

function TenureAnalysis({ positions }) {
    const tenures = positions.map(p => p.tenureMonths);
    const avgTenure = tenures.length > 0
        ? Math.round(tenures.reduce((a, b) => a + b, 0) / tenures.length)
        : 0;
    const maxTenure = Math.max(...tenures, 0);
    const minTenure = Math.min(...tenures.filter(t => t > 0), 0);

    const chartData = positions.map(p => ({
        name: p.company.length > 20 ? p.company.substring(0, 20) + '...' : p.company,
        months: p.tenureMonths,
        current: p.isCurrent
    })).reverse();

    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <h3 className="text-xl font-semibold mb-4">Tenure Analysis</h3>

            <div className="grid grid-cols-3 gap-4 mb-6">
                <div className="text-center p-4 bg-gray-700/50 rounded-lg">
                    <p className="text-2xl font-bold text-white">
                        {Math.floor(avgTenure / 12)}y {avgTenure % 12}m
                    </p>
                    <p className="text-sm text-gray-400">Average Tenure</p>
                </div>
                <div className="text-center p-4 bg-gray-700/50 rounded-lg">
                    <p className="text-2xl font-bold text-green-400">
                        {Math.floor(maxTenure / 12)}y {maxTenure % 12}m
                    </p>
                    <p className="text-sm text-gray-400">Longest Role</p>
                </div>
                <div className="text-center p-4 bg-gray-700/50 rounded-lg">
                    <p className="text-2xl font-bold text-yellow-400">
                        {Math.floor(minTenure / 12)}y {minTenure % 12}m
                    </p>
                    <p className="text-sm text-gray-400">Shortest Role</p>
                </div>
            </div>

            <ResponsiveContainer width="100%" height={200}>
                <BarChart data={chartData}>
                    <XAxis dataKey="name" stroke="#666" tick={{ fontSize: 10 }} angle={-45} textAnchor="end" height={80} />
                    <YAxis stroke="#666" label={{ value: 'Months', angle: -90, position: 'insideLeft', fill: '#666' }} />
                    <Tooltip
                        contentStyle={{ backgroundColor: '#1f2937', border: 'none', borderRadius: '8px' }}
                        formatter={(value) => [`${Math.floor(value/12)}y ${value%12}m`, 'Tenure']}
                    />
                    <Bar dataKey="months" fill="#0A66C2" radius={[4, 4, 0, 0]}>
                        {chartData.map((entry, index) => (
                            <Cell key={index} fill={entry.current ? '#22c55e' : '#0A66C2'} />
                        ))}
                    </Bar>
                </BarChart>
            </ResponsiveContainer>
        </div>
    );
}

function CareerInsights({ positions }) {
    // Calculate career stats
    const totalExperience = positions.reduce((sum, p) => sum + p.tenureMonths, 0);
    const uniqueCompanies = new Set(positions.map(p => p.company)).size;
    const currentRole = positions.find(p => p.isCurrent);

    // Detect career transitions
    const transitions = [];
    for (let i = 0; i < positions.length - 1; i++) {
        const current = positions[i];
        const prev = positions[i + 1];

        // Simple transition detection based on title keywords
        const currentType = current.title?.toLowerCase().includes('founder') ? 'Founder' :
                          current.title?.toLowerCase().includes('director') ? 'Director' :
                          current.title?.toLowerCase().includes('manager') ? 'Manager' :
                          current.title?.toLowerCase().includes('lead') ? 'Lead' :
                          current.title?.toLowerCase().includes('consultant') ? 'Consultant' :
                          'Individual Contributor';
        const prevType = prev.title?.toLowerCase().includes('founder') ? 'Founder' :
                        prev.title?.toLowerCase().includes('director') ? 'Director' :
                        prev.title?.toLowerCase().includes('manager') ? 'Manager' :
                        prev.title?.toLowerCase().includes('lead') ? 'Lead' :
                        prev.title?.toLowerCase().includes('consultant') ? 'Consultant' :
                        'Individual Contributor';

        if (currentType !== prevType) {
            transitions.push({ from: prevType, to: currentType, year: current.startDate?.getFullYear() });
        }
    }

    return (
        <div className="bg-gray-800 rounded-xl p-6">
            <h3 className="text-xl font-semibold mb-4">Career Insights</h3>

            <div className="space-y-4">
                <div className="flex justify-between items-center p-3 bg-gray-700/50 rounded-lg">
                    <span className="text-gray-300">Total Experience</span>
                    <span className="font-semibold">{Math.floor(totalExperience / 12)} years {totalExperience % 12} months</span>
                </div>

                <div className="flex justify-between items-center p-3 bg-gray-700/50 rounded-lg">
                    <span className="text-gray-300">Companies Worked At</span>
                    <span className="font-semibold">{uniqueCompanies}</span>
                </div>

                <div className="flex justify-between items-center p-3 bg-gray-700/50 rounded-lg">
                    <span className="text-gray-300">Current Role</span>
                    <span className="font-semibold text-green-400">{currentRole?.title || 'N/A'}</span>
                </div>

                {transitions.length > 0 && (
                    <div className="mt-4">
                        <p className="text-gray-400 text-sm mb-2">Career Transitions</p>
                        <div className="flex flex-wrap gap-2">
                            {transitions.map((t, i) => (
                                <span key={i} className="bg-linkedin/20 text-linkedin px-3 py-1 rounded-full text-sm">
                                    {t.from} → {t.to} ({t.year})
                                </span>
                            ))}
                        </div>
                    </div>
                )}
            </div>
        </div>
    );
}

function CareerTrajectory({ data, insights }) {
    return (
        <div className="space-y-6">
            <h2 className="text-2xl font-bold flex items-center gap-2">
                <span>📈</span> Career Trajectory
            </h2>

            <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                <StatCard
                    label="Total Positions"
                    value={insights.trajectory.length}
                />
                <StatCard
                    label="Years in Career"
                    value={Math.floor(insights.trajectory.reduce((s, p) => s + p.tenureMonths, 0) / 12)}
                />
                <StatCard
                    label="Current Role"
                    value={insights.trajectory.find(p => p.isCurrent)?.title?.split(' ')[0] || 'N/A'}
                />
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                <CareerTimeline positions={insights.trajectory} />
                <div className="space-y-6">
                    <TenureAnalysis positions={insights.trajectory} />
                    <CareerInsights positions={insights.trajectory} />
                </div>
            </div>
        </div>
    );
}
```

**Step 2: Verify no syntax errors**

Run: `open linkedin-insights.html`
Expected: Page loads without JavaScript errors

**Step 3: Commit**

```bash
git add linkedin-insights.html
git commit -m "feat: add Career Trajectory dashboard components"
```

---

## Task 6: Wire Up Dashboard and Complete App

**Files:**
- Modify: `linkedin-insights.html`

**Step 1: Add Dashboard component and update App**

Add Dashboard component and update App:

```jsx
function Dashboard({ data }) {
    const [activeTab, setActiveTab] = useState('network');

    // Compute all insights
    const userProfileUrl = 'https://www.linkedin.com/in/kyraatekwana'; // Will be dynamic in real use
    const messagePartners = DataUtils.getMessagePartners(data.messages || [], userProfileUrl);

    const insights = {
        composition: DataUtils.getNetworkComposition(data.connections || []),
        growth: DataUtils.getConnectionGrowth(data.connections || []),
        dormant: DataUtils.getDormantConnections(data.connections || [], messagePartners, 12),
        strategic: DataUtils.getStrategicConnections(
            data.connections || [],
            data.companyFollows || [],
            data.savedJobs || [],
            messagePartners
        ),
        trajectory: DataUtils.getCareerTrajectory(data.positions || [])
    };

    return (
        <div className="max-w-7xl mx-auto">
            {/* Tab Navigation */}
            <div className="flex justify-center gap-4 mb-8">
                <button
                    onClick={() => setActiveTab('network')}
                    className={`px-6 py-3 rounded-full font-medium transition ${
                        activeTab === 'network'
                            ? 'bg-linkedin text-white'
                            : 'bg-gray-800 text-gray-300 hover:bg-gray-700'
                    }`}
                >
                    🔗 Network Health
                </button>
                <button
                    onClick={() => setActiveTab('career')}
                    className={`px-6 py-3 rounded-full font-medium transition ${
                        activeTab === 'career'
                            ? 'bg-linkedin text-white'
                            : 'bg-gray-800 text-gray-300 hover:bg-gray-700'
                    }`}
                >
                    📈 Career Trajectory
                </button>
            </div>

            {/* Content */}
            {activeTab === 'network' && <NetworkHealth data={data} insights={insights} />}
            {activeTab === 'career' && <CareerTrajectory data={data} insights={insights} />}

            {/* Reset Button */}
            <div className="mt-12 text-center">
                <button
                    onClick={() => window.location.reload()}
                    className="text-gray-500 hover:text-gray-300 text-sm"
                >
                    ↩ Upload different data
                </button>
            </div>

            {/* Footer */}
            <footer className="mt-12 text-center text-gray-600 text-sm">
                <p>Your data never leaves your browser. Built with privacy in mind.</p>
            </footer>
        </div>
    );
}

function App() {
    const [data, setData] = useState(null);

    if (!data) {
        return (
            <div className="min-h-screen p-8">
                <h1 className="text-4xl font-bold text-center mb-2">LinkedIn Insights</h1>
                <p className="text-center text-gray-400 mb-12">Discover actionable insights from your professional network</p>
                <FileUpload onDataLoaded={setData} />
            </div>
        );
    }

    return (
        <div className="min-h-screen p-8">
            <h1 className="text-4xl font-bold text-center mb-2">LinkedIn Insights</h1>
            <p className="text-center text-gray-400 mb-8">
                Analyzing {data.connections?.length?.toLocaleString() || 0} connections
            </p>
            <Dashboard data={data} />
        </div>
    );
}

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
```

**Step 2: Test full flow with real LinkedIn data**

Run: `open linkedin-insights.html`
Action: Upload `~/Downloads/Basic_LinkedInDataExport_11-29-2025.zip`
Expected:
- Dashboard renders with two tabs
- Network Health shows connections, charts, dormant and strategic lists
- Career Trajectory shows timeline and tenure analysis
- Clicking on dormant/strategic connections opens LinkedIn profiles

**Step 3: Commit**

```bash
git add linkedin-insights.html
git commit -m "feat: wire up Dashboard with tabs and complete data flow"
```

---

## Task 7: Polish UI and Add Final Touches

**Files:**
- Modify: `linkedin-insights.html`

**Step 1: Add loading states, better error handling, and visual polish**

Update the styles section in the `<head>`:

```html
<style>
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; }
    .gradient-bg { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); }
    .glass { background: rgba(31, 41, 55, 0.5); backdrop-filter: blur(10px); }

    /* Custom scrollbar */
    ::-webkit-scrollbar { width: 8px; height: 8px; }
    ::-webkit-scrollbar-track { background: #1f2937; border-radius: 4px; }
    ::-webkit-scrollbar-thumb { background: #4b5563; border-radius: 4px; }
    ::-webkit-scrollbar-thumb:hover { background: #6b7280; }

    /* Animations */
    @keyframes fadeIn {
        from { opacity: 0; transform: translateY(10px); }
        to { opacity: 1; transform: translateY(0); }
    }
    .fade-in { animation: fadeIn 0.3s ease-out forwards; }

    @keyframes pulse {
        0%, 100% { opacity: 1; }
        50% { opacity: 0.5; }
    }
    .animate-pulse { animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite; }
</style>
```

**Step 2: Add a header with branding**

Update the App render for non-data state:

```jsx
function Header() {
    return (
        <header className="text-center mb-12">
            <div className="inline-flex items-center gap-3 mb-4">
                <div className="w-12 h-12 bg-linkedin rounded-xl flex items-center justify-center">
                    <span className="text-2xl">📊</span>
                </div>
                <h1 className="text-4xl font-bold">LinkedIn Insights</h1>
            </div>
            <p className="text-gray-400 max-w-xl mx-auto">
                Discover actionable insights from your professional network.
                Find dormant connections to reactivate and strategic paths to your target companies.
            </p>
        </header>
    );
}
```

**Step 3: Test visual polish**

Run: `open linkedin-insights.html`
Expected: Improved visuals, smooth scrolling, nice animations

**Step 4: Commit**

```bash
git add linkedin-insights.html
git commit -m "feat: add visual polish, animations, and improved UX"
```

---

## Task 8: Final Testing and Documentation

**Files:**
- Modify: `linkedin-insights.html` (add inline help)
- Create: `README.md`

**Step 1: Add help tooltip to upload area**

Add to FileUpload component:

```jsx
<details className="mt-6 text-left text-sm text-gray-500">
    <summary className="cursor-pointer hover:text-gray-400">How to get your LinkedIn data export</summary>
    <ol className="mt-2 ml-4 space-y-1 list-decimal">
        <li>Go to <a href="https://www.linkedin.com/mypreferences/d/download-my-data" target="_blank" className="text-linkedin hover:underline">LinkedIn Data Export</a></li>
        <li>Select "Want something in particular?" and check the boxes you need</li>
        <li>Click "Request archive"</li>
        <li>Wait for LinkedIn's email (can take up to 24 hours)</li>
        <li>Download and upload the ZIP here</li>
    </ol>
</details>
```

**Step 2: Create README.md**

```markdown
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
```

**Step 3: Final test with fresh browser**

Run: `open -a "Google Chrome" --args --incognito linkedin-insights.html`
Expected: Full flow works from upload to insights display

**Step 4: Final commit**

```bash
git add linkedin-insights.html README.md
git commit -m "docs: add inline help and README"
```

---

## Summary

This plan creates a complete LinkedIn Insights tool in 8 tasks:

1. **Base HTML Shell** - Set up dependencies
2. **File Upload** - ZIP extraction and CSV parsing
3. **Data Utilities** - Processing functions for insights
4. **Network Health** - Composition, growth, dormant, strategic
5. **Career Trajectory** - Timeline, tenure, insights
6. **Dashboard Wiring** - Connect everything with tabs
7. **Visual Polish** - Animations, scrollbars, branding
8. **Documentation** - Help text and README

Total: ~1 HTML file, ~800 lines of code, fully self-contained.
