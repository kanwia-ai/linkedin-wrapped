# Intent Mode & Forgotten VIPs Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Intent Mode toggle and Forgotten VIPs to replace overwhelming dormant connections list with a curated, actionable shortlist.

**Architecture:** Add intent state at App level, pass down to Dashboard. Create scoring system for VIP prioritization. Replace DormantConnections component with ForgottenVIPs that shows top 15-20 high-signal connections based on intent + signals.

**Tech Stack:** React (existing), Tailwind (existing), no new dependencies

---

## Task 1: Add Intent Mode State and Selector Component

**Files:**
- Modify: `linkedin-insights.html` (lines 753-824, Dashboard component area)

**Step 1: Create IntentModeSelector component**

Add after the COLORS array (around line 307), before StatCard:

```jsx
const INTENT_MODES = {
    general: {
        id: 'general',
        label: 'General Networking',
        icon: '🤝',
        description: 'Maintain and grow professional relationships'
    },
    jobSearch: {
        id: 'jobSearch',
        label: 'Active Job Search',
        icon: '💼',
        description: 'Find connections at target companies'
    },
    building: {
        id: 'building',
        label: 'Building in a Space',
        icon: '🚀',
        description: 'Connect with founders, operators, experts'
    },
    fundraising: {
        id: 'fundraising',
        label: 'Fundraising/Partnerships',
        icon: '💰',
        description: 'Find investors and potential partners'
    }
};

function IntentModeSelector({ intent, onIntentChange }) {
    return (
        <div className="bg-gray-800 rounded-xl p-4 mb-6 border border-gray-700">
            <p className="text-sm text-gray-400 mb-3 font-medium">What are you optimizing for?</p>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-2">
                {Object.values(INTENT_MODES).map(mode => (
                    <button
                        key={mode.id}
                        onClick={() => onIntentChange(mode.id)}
                        className={`p-3 rounded-lg text-left transition-all ${
                            intent === mode.id
                                ? 'bg-linkedin/20 border-2 border-linkedin'
                                : 'bg-gray-700/50 border-2 border-transparent hover:bg-gray-700'
                        }`}
                    >
                        <div className="text-xl mb-1">{mode.icon}</div>
                        <div className="text-sm font-medium">{mode.label}</div>
                    </button>
                ))}
            </div>
        </div>
    );
}
```

**Step 2: Add intent state to Dashboard component**

In Dashboard function (line 753), add state:

```jsx
function Dashboard({ data, userProfileUrl }) {
    const [activeTab, setActiveTab] = useState('network');
    const [insights, setInsights] = useState(null);
    const [intent, setIntent] = useState('general'); // NEW
```

**Step 3: Add IntentModeSelector to Dashboard render**

After the tab navigation div and before the content div (around line 804):

```jsx
                    </div>

                    {/* Intent Mode Selector - NEW */}
                    <IntentModeSelector intent={intent} onIntentChange={setIntent} />

                    {/* Content */}
```

**Step 4: Verify page loads without errors**

Run: `open ~/Desktop/linkedin-data/linkedin-insights.html`
Expected: Intent mode selector appears between tabs and content

**Step 5: Commit**

```bash
cd ~/Desktop/linkedin-data && git add linkedin-insights.html && git commit -m "feat: add Intent Mode selector component"
```

---

## Task 2: Create VIP Scoring System in DataUtils

**Files:**
- Modify: `linkedin-insights.html` (DataUtils object, lines 75-303)

**Step 1: Add getSeniorityScore helper function**

Add to DataUtils object (after getCareerTrajectory, before the closing brace):

```jsx
            // Score seniority from title (0-100)
            getSeniorityScore(title) {
                if (!title) return 0;
                const t = title.toLowerCase();

                // C-suite and founders
                if (t.includes('ceo') || t.includes('cto') || t.includes('cfo') ||
                    t.includes('chief') || t.includes('founder') || t.includes('co-founder') ||
                    t.includes('president')) return 100;

                // VP level
                if (t.includes('vp') || t.includes('vice president')) return 90;

                // Director/Head level
                if (t.includes('director') || t.includes('head of') || t.includes('head,')) return 80;

                // Senior Manager/Lead
                if (t.includes('senior manager') || t.includes('principal') ||
                    t.includes('lead') || t.includes('staff')) return 70;

                // Manager level
                if (t.includes('manager')) return 60;

                // Senior IC
                if (t.includes('senior') || t.includes('sr.') || t.includes('sr ')) return 50;

                // Standard IC
                return 30;
            },

            // Check if connection shares history with user (same school/company)
            sharesHistory(connection, userPositions, userEducation) {
                const connCompany = (connection.company || '').toLowerCase();
                const userCompanies = (userPositions || []).map(p => (p['Company Name'] || '').toLowerCase());

                // Check if they worked at same company
                if (userCompanies.some(c => c && connCompany.includes(c))) return true;

                // Could add education check if we had that data structured
                return false;
            },

            // Get Forgotten VIPs - high-signal dormant connections
            getForgottenVIPs(dormantConnections, data, intent, limit = 20) {
                const targetCompanies = new Set();

                // Build target company set from followed companies and saved jobs
                (data.companyFollows || []).forEach(f => {
                    if (f['Organization']) targetCompanies.add(f['Organization'].toLowerCase());
                });
                (data.savedJobs || []).forEach(j => {
                    if (j['Company Name']) targetCompanies.add(j['Company Name'].toLowerCase());
                });

                // Score each dormant connection
                const scored = dormantConnections.map(conn => {
                    let score = 0;
                    const reasons = [];

                    // Seniority score (0-100, weighted)
                    const seniorityScore = this.getSeniorityScore(conn.position);
                    score += seniorityScore * 0.3;
                    if (seniorityScore >= 80) reasons.push('Senior role');

                    // At target company (big boost)
                    const isAtTarget = targetCompanies.has((conn.company || '').toLowerCase());
                    if (isAtTarget) {
                        score += 40;
                        reasons.push('At target company');
                    }

                    // Shared history (would need user positions passed in)
                    if (this.sharesHistory(conn, data.positions, data.education)) {
                        score += 20;
                        reasons.push('Shared history');
                    }

                    // Recency of connection (more recent = warmer)
                    if (conn.connectedOn) {
                        const monthsAgo = Math.floor((new Date() - conn.connectedOn) / (1000 * 60 * 60 * 24 * 30));
                        if (monthsAgo < 24) {
                            score += 15;
                            reasons.push('Connected recently');
                        } else if (monthsAgo < 48) {
                            score += 10;
                        }
                    }

                    // Intent-specific boosts
                    const position = (conn.position || '').toLowerCase();
                    const company = (conn.company || '').toLowerCase();

                    if (intent === 'jobSearch') {
                        // Boost recruiters and hiring managers
                        if (position.includes('recruit') || position.includes('talent') ||
                            position.includes('hr') || position.includes('people')) {
                            score += 25;
                            reasons.push('Talent/HR role');
                        }
                        if (isAtTarget) score += 20; // Extra boost for target companies
                    }

                    if (intent === 'fundraising') {
                        // Boost investors and partners
                        if (position.includes('investor') || position.includes('partner') ||
                            position.includes('venture') || position.includes('capital') ||
                            company.includes('venture') || company.includes('capital')) {
                            score += 35;
                            reasons.push('Investor/VC');
                        }
                    }

                    if (intent === 'building') {
                        // Boost founders and operators
                        if (position.includes('founder') || position.includes('ceo') ||
                            position.includes('operator')) {
                            score += 25;
                            reasons.push('Founder/Operator');
                        }
                    }

                    return {
                        ...conn,
                        vipScore: Math.round(score),
                        vipReasons: reasons
                    };
                });

                // Sort by score and return top N
                return scored
                    .filter(c => c.vipScore > 20) // Only include if has some signal
                    .sort((a, b) => b.vipScore - a.vipScore)
                    .slice(0, limit);
            }
```

**Step 2: Verify no syntax errors**

Run: `open ~/Desktop/linkedin-data/linkedin-insights.html`
Expected: Page loads without JavaScript errors

**Step 3: Commit**

```bash
cd ~/Desktop/linkedin-data && git add linkedin-insights.html && git commit -m "feat: add VIP scoring system to DataUtils"
```

---

## Task 3: Create ForgottenVIPs Component

**Files:**
- Modify: `linkedin-insights.html` (add new component after DormantConnections)

**Step 1: Add ForgottenVIPs component**

Add after DormantConnections component (around line 425):

```jsx
function ForgottenVIPs({ vips, intent }) {
    if (!vips || vips.length === 0) {
        return (
            <div className="bg-gray-800 rounded-xl p-6 shadow-lg border border-gray-700">
                <h3 className="text-xl font-semibold flex items-center gap-2">
                    <span>⭐</span> Forgotten VIPs
                </h3>
                <p className="text-gray-400 mt-2">
                    No high-priority dormant connections found for your current intent.
                    Try switching intent modes or following more companies.
                </p>
            </div>
        );
    }

    return (
        <div className="bg-gray-800 rounded-xl p-6 shadow-lg border border-gray-700">
            <div className="flex justify-between items-center mb-4">
                <div>
                    <h3 className="text-xl font-semibold flex items-center gap-2">
                        <span>⭐</span> Forgotten VIPs
                    </h3>
                    <p className="text-gray-400 text-sm">
                        High-value connections worth re-engaging
                    </p>
                </div>
                <span className="bg-purple-600/30 text-purple-400 px-3 py-1 rounded-full text-sm font-medium">
                    Top {vips.length}
                </span>
            </div>

            <div className="space-y-3 max-h-[500px] overflow-y-auto pr-2">
                {vips.map((vip, i) => (
                    <div
                        key={i}
                        className="p-4 bg-gray-700/50 rounded-lg hover:bg-gray-700 transition border-l-4 border-purple-500"
                    >
                        <div className="flex justify-between items-start mb-2">
                            <div className="flex-1 min-w-0">
                                <a
                                    href={vip.url}
                                    target="_blank"
                                    rel="noopener noreferrer"
                                    className="font-semibold text-lg hover:text-linkedin truncate block"
                                >
                                    {vip.name}
                                </a>
                                <p className="text-gray-300">{vip.position}</p>
                                <p className="text-linkedin text-sm">{vip.company}</p>
                            </div>
                            <div className="text-right shrink-0 ml-4">
                                <div className="text-2xl font-bold text-purple-400">{vip.vipScore}</div>
                                <div className="text-xs text-gray-500">VIP Score</div>
                            </div>
                        </div>

                        {/* Reason tags */}
                        {vip.vipReasons.length > 0 && (
                            <div className="flex flex-wrap gap-1 mt-2">
                                {vip.vipReasons.map((reason, j) => (
                                    <span
                                        key={j}
                                        className="px-2 py-0.5 bg-gray-600/50 rounded text-xs text-gray-300"
                                    >
                                        {reason}
                                    </span>
                                ))}
                            </div>
                        )}

                        {/* Context */}
                        <div className="mt-3 pt-3 border-t border-gray-600 flex justify-between text-xs text-gray-500">
                            <span>
                                Connected: {vip.connectedOn?.toLocaleDateString('en-US', { month: 'short', year: 'numeric' }) || 'Unknown'}
                            </span>
                            <span>
                                {vip.lastMessage
                                    ? `Last msg: ${vip.lastMessage.toLocaleDateString('en-US', { month: 'short', year: 'numeric' })}`
                                    : 'Never messaged'
                                }
                            </span>
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

**Step 2: Verify component renders (manual check)**

Expected: No errors, component defined

**Step 3: Commit**

```bash
cd ~/Desktop/linkedin-data && git add linkedin-insights.html && git commit -m "feat: add ForgottenVIPs component with VIP cards"
```

---

## Task 4: Integrate VIPs into Dashboard and NetworkHealth

**Files:**
- Modify: `linkedin-insights.html` (Dashboard useEffect, NetworkHealth component)

**Step 1: Update Dashboard useEffect to compute VIPs**

In Dashboard's useEffect (around line 757), update the calculated object:

```jsx
            useEffect(() => {
                if(!data) return;

                // Compute all insights
                const messagePartners = DataUtils.getMessagePartners(data.messages || [], userProfileUrl);
                const dormant = DataUtils.getDormantConnections(data.connections || [], messagePartners, 12);

                const calculated = {
                    composition: DataUtils.getNetworkComposition(data.connections || []),
                    growth: DataUtils.getConnectionGrowth(data.connections || []),
                    dormant: dormant,
                    forgottenVIPs: DataUtils.getForgottenVIPs(dormant, data, intent, 20), // NEW
                    strategic: DataUtils.getStrategicConnections(
                        data.connections || [],
                        data.companyFollows || [],
                        data.savedJobs || [],
                        messagePartners
                    ),
                    trajectory: DataUtils.getCareerTrajectory(data.positions || [])
                };
                setInsights(calculated);
            }, [data, userProfileUrl, intent]); // Add intent to dependencies
```

**Step 2: Pass intent to NetworkHealth**

Update the NetworkHealth call (around line 808):

```jsx
                        {activeTab === 'network' && <NetworkHealth data={data} insights={insights} intent={intent} />}
```

**Step 3: Update NetworkHealth to use ForgottenVIPs**

Update NetworkHealth component signature and content (around line 495):

```jsx
        function NetworkHealth({ data, insights, intent }) {
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
                            label="Forgotten VIPs"
                            value={insights.forgottenVIPs?.length || 0}
                            subtitle="high-priority"
                        />
                        <StatCard
                            label="Strategic"
                            value={insights.strategic.length}
                            subtitle="at target companies"
                        />
                    </div>

                    <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                        <NetworkComposition data={insights.composition.topCompanies} />
                        <ConnectionGrowth data={insights.growth} />
                    </div>

                    <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                        <ForgottenVIPs vips={insights.forgottenVIPs} intent={intent} />
                        <StrategicConnections connections={insights.strategic} />
                    </div>
                </div>
            );
        }
```

**Step 4: Test the full integration**

Run: `open ~/Desktop/linkedin-data/linkedin-insights.html`
Upload LinkedIn data, verify:
- Intent selector appears and works
- ForgottenVIPs shows curated list with scores
- Changing intent updates the VIP list
- No more "Show all 1948" button

**Step 5: Commit**

```bash
cd ~/Desktop/linkedin-data && git add linkedin-insights.html && git commit -m "feat: integrate ForgottenVIPs into Dashboard with intent-based filtering"
```

---

## Task 5: Add Collapsible "All Dormant" Section

**Files:**
- Modify: `linkedin-insights.html` (NetworkHealth component)

**Step 1: Add collapsible dormant section below main grid**

In NetworkHealth, after the ForgottenVIPs/Strategic grid, add:

```jsx
                    {/* Collapsible full dormant list */}
                    <details className="bg-gray-800 rounded-xl border border-gray-700">
                        <summary className="p-4 cursor-pointer hover:bg-gray-700/50 transition flex justify-between items-center">
                            <span className="text-gray-400">
                                View all {insights.dormant.length.toLocaleString()} dormant connections
                            </span>
                            <span className="text-gray-500 text-sm">
                                (12+ months no contact)
                            </span>
                        </summary>
                        <div className="p-4 pt-0 max-h-96 overflow-y-auto">
                            <div className="space-y-2">
                                {insights.dormant.slice(0, 100).map((conn, i) => (
                                    <a
                                        key={i}
                                        href={conn.url}
                                        target="_blank"
                                        rel="noopener noreferrer"
                                        className="flex justify-between items-center p-2 hover:bg-gray-700/50 rounded text-sm"
                                    >
                                        <span className="text-gray-300 truncate">{conn.name}</span>
                                        <span className="text-gray-500 truncate ml-2">{conn.company}</span>
                                    </a>
                                ))}
                                {insights.dormant.length > 100 && (
                                    <p className="text-center text-gray-500 text-sm py-2">
                                        + {insights.dormant.length - 100} more...
                                    </p>
                                )}
                            </div>
                        </div>
                    </details>
```

**Step 2: Test collapsible section**

Run: `open ~/Desktop/linkedin-data/linkedin-insights.html`
Expected: Collapsed by default, expands to show dormant list

**Step 3: Commit**

```bash
cd ~/Desktop/linkedin-data && git add linkedin-insights.html && git commit -m "feat: add collapsible section for full dormant list"
```

---

## Task 6: Final Polish and Testing

**Files:**
- Modify: `linkedin-insights.html` (minor adjustments)

**Step 1: Remove old DormantConnections from NetworkHealth grid**

The old DormantConnections component is no longer used in NetworkHealth. We can keep it defined (might be useful) but it's now replaced by ForgottenVIPs.

**Step 2: Full integration test**

Run: `open ~/Desktop/linkedin-data/linkedin-insights.html`

Test checklist:
- [ ] Intent selector shows 4 options with icons
- [ ] Clicking intent updates selection visually
- [ ] ForgottenVIPs section shows top ~20 connections
- [ ] VIP cards show name, position, company, score, reason tags
- [ ] Changing intent changes which VIPs appear (esp. job search vs general)
- [ ] Collapsible "View all dormant" works
- [ ] Strategic connections still works
- [ ] Charts still render
- [ ] Career Trajectory tab still works
- [ ] No console errors

**Step 3: Final commit**

```bash
cd ~/Desktop/linkedin-data && git add linkedin-insights.html && git commit -m "feat: complete Intent Mode and Forgotten VIPs implementation"
```

---

## Summary

This plan adds two major features in 6 tasks:

| Task | What It Builds |
|------|----------------|
| 1 | Intent Mode selector component + state |
| 2 | VIP scoring system in DataUtils |
| 3 | ForgottenVIPs component with rich cards |
| 4 | Wire VIPs into Dashboard with intent filtering |
| 5 | Collapsible section for full dormant list |
| 6 | Polish and testing |

**Key behavior changes:**
- User selects intent at top of dashboard
- "Forgotten VIPs" replaces overwhelming dormant list
- VIPs are scored based on seniority, target companies, shared history, recency, and intent-specific boosts
- Full dormant list still accessible but hidden by default

**Ready for Gemini:** The VIP reasons/scoring are rule-based. You can later enhance with Gemini by passing the VIP list to Gemini for AI-generated context like "You met at Booth in 2020, she's now leading AI at Notion."
