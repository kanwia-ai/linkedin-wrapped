# Career Actions & Relationship Intel Tabs Implementation Plan

**Goal:** Add two new tabs with conversational coaching insights - "curious friend" voice that observes patterns and prompts reflection.

**Tab Structure:** `Network Health` | `Career Actions` | `Relationship Intel` | `Career Trajectory`

**Voice Examples:**
- "Noticed you've averaged 1.2 years per role. Not judging! But interviewers might ask. Have you thought about how you'd frame each move?"
- "Your top 5 most-messaged connections are all from your 2019-2021 era. Your active network might be getting a bit stale?"

---

## Task 1: Update Tab Navigation

**Files:** `linkedin-insights.html` (Dashboard component)

**Step 1:** Update tab buttons in Dashboard to show 4 tabs:
```jsx
const tabs = [
    { id: 'network', label: 'Network Health', icon: '🔗' },
    { id: 'careerActions', label: 'Career Actions', icon: '🎯' },
    { id: 'relationshipIntel', label: 'Relationship Intel', icon: '💬' },
    { id: 'career', label: 'Career Trajectory', icon: '📈' }
];
```

**Step 2:** Update content rendering to include new tabs

**Step 3:** Move IntentModeSelector to only show on 'network' tab (already done)

---

## Task 2: Create Career Insights Computation in DataUtils

**Files:** `linkedin-insights.html` (DataUtils object)

**Add function `getCareerInsights(positions, jobApplications, savedJobs)`:**

```jsx
getCareerInsights(positions, jobApplications, savedJobs) {
    const insights = [];

    // 1. Tenure Analysis
    const tenures = positions.map(p => p.tenureMonths).filter(t => t > 0);
    const avgTenure = tenures.reduce((a, b) => a + b, 0) / tenures.length;
    const shortStints = positions.filter(p => p.tenureMonths < 12).length;

    if (avgTenure < 18) {
        insights.push({
            id: 'short-tenure',
            type: 'tenure',
            icon: '⏱️',
            headline: `Your average role tenure is ${Math.floor(avgTenure / 12)}y ${Math.round(avgTenure % 12)}m`,
            observation: `You've had ${shortStints} roles under a year. That's not necessarily bad - maybe you were exploring, or startups imploded, or you got poached. But interviewers will definitely ask.`,
            prompt: `Have you thought about your narrative for quick moves? The best answer shows intentionality, not restlessness.`,
            action: 'Prep your story'
        });
    } else if (avgTenure > 48) {
        insights.push({
            id: 'long-tenure',
            type: 'tenure',
            icon: '🏠',
            headline: `You've averaged ${Math.floor(avgTenure / 12)}+ years per role`,
            observation: `That's solid tenure - shows commitment. But if you're job searching now, be ready for "why leave after so long?" questions.`,
            prompt: `What's pulling you toward something new? Growth ceiling? New challenge? That's your answer.`,
            action: 'Frame your "why now"'
        });
    }

    // 2. Job Search Behavior
    const applied = (jobApplications || []).length;
    const saved = (savedJobs || []).length;

    if (applied > 0) {
        const ratio = saved > 0 ? Math.round((applied / saved) * 100) : null;

        if (ratio && ratio < 30) {
            insights.push({
                id: 'low-apply-rate',
                type: 'job-search',
                icon: '🎯',
                headline: `You've saved ${saved} jobs but only applied to ${applied}`,
                observation: `That's a ${ratio}% save-to-apply rate. Either you're super selective (good!), or there's friction stopping you from pulling the trigger.`,
                prompt: `What's holding you back on the saved ones? Requirements you don't meet? Cover letter fatigue? Imposter syndrome?`,
                action: 'Review saved jobs'
            });
        } else if (applied > 50) {
            insights.push({
                id: 'high-volume-apply',
                type: 'job-search',
                icon: '📨',
                headline: `You've applied to ${applied} jobs`,
                observation: `That's a lot of applications! Are you getting responses? High volume can work, but targeted apps with warm intros usually convert better.`,
                prompt: `Have you looked at which connections could refer you to your target companies?`,
                action: 'Find warm intros'
            });
        }
    }

    // 3. Career Progression
    const getLevel = (title) => {
        const t = (title || '').toLowerCase();
        if (t.includes('founder') || t.includes('ceo') || t.includes('chief') || t.includes('president')) return 5;
        if (t.includes('vp') || t.includes('vice president') || t.includes('director')) return 4;
        if (t.includes('senior manager') || t.includes('head of')) return 3;
        if (t.includes('manager') || t.includes('lead')) return 2;
        if (t.includes('senior') || t.includes('sr')) return 1;
        return 0;
    };

    const levelHistory = positions.map(p => ({ level: getLevel(p.title), title: p.title, year: p.startDate?.getFullYear() }));
    const currentLevel = levelHistory[0]?.level || 0;
    const peakLevel = Math.max(...levelHistory.map(l => l.level));

    if (currentLevel < peakLevel) {
        insights.push({
            id: 'level-dip',
            type: 'progression',
            icon: '📉',
            headline: `You've held more senior titles before`,
            observation: `Looks like you stepped back from a higher level role at some point. Totally valid - maybe you switched industries, joined a startup, or prioritized learning over title.`,
            prompt: `Just be ready to explain it if asked. "I traded title for [growth/equity/impact]" usually lands well.`,
            action: 'Prep your narrative'
        });
    }

    // Check for plateau
    const recentRoles = positions.slice(0, 3);
    const sameLevelCount = recentRoles.filter(p => getLevel(p.title) === currentLevel).length;
    if (sameLevelCount >= 3 && positions.length >= 3) {
        const years = Math.round((new Date() - positions[2]?.startDate) / (1000 * 60 * 60 * 24 * 365));
        if (years >= 4) {
            insights.push({
                id: 'plateau',
                type: 'progression',
                icon: '📊',
                headline: `You've been at a similar level for ${years}+ years`,
                observation: `Your last 3 roles have been around the same seniority. That's not bad if you're deepening expertise! But if you're itching for growth...`,
                prompt: `Is the next level what you actually want? Sometimes lateral moves to bigger scope are smarter than chasing titles.`,
                action: 'Define your growth goal'
            });
        }
    }

    // 4. Industry/Company Patterns
    const companies = positions.map(p => p.company);
    const uniqueCompanies = new Set(companies).size;

    if (uniqueCompanies <= 2 && positions.length >= 4) {
        insights.push({
            id: 'company-loyal',
            type: 'pattern',
            icon: '🏢',
            headline: `You've mostly stayed at ${uniqueCompanies} company${uniqueCompanies > 1 ? 'ies' : ''}`,
            observation: `Multiple roles at the same company shows growth and trust. But it can also mean your external network is smaller than peers who've moved around.`,
            prompt: `When you do look externally, lean on your connections from those companies - they know your work.`,
            action: 'Leverage internal advocates'
        });
    }

    return insights;
}
```

---

## Task 3: Create Relationship Insights Computation in DataUtils

**Files:** `linkedin-insights.html` (DataUtils object)

**Add function `getRelationshipInsights(messages, connections, messagePartners, savedJobs, companyFollows, dormant, strategic)`:**

```jsx
getRelationshipInsights(data, messagePartners, dormant, strategic, founders, recruiters, investors, executives) {
    const insights = [];
    const now = new Date();

    // 1. Message Recency - Top messaged people era
    const partners = Object.values(messagePartners);
    const topPartners = partners.sort((a, b) => b.messageCount - a.messageCount).slice(0, 10);

    if (topPartners.length > 0) {
        const avgYear = topPartners.reduce((sum, p) => {
            return sum + (p.lastMessage?.getFullYear() || now.getFullYear());
        }, 0) / topPartners.length;

        const yearsAgo = now.getFullYear() - Math.round(avgYear);

        if (yearsAgo >= 2) {
            insights.push({
                id: 'stale-active-network',
                type: 'recency',
                icon: '📅',
                headline: `Your most-messaged connections are from ${Math.round(avgYear)}-ish`,
                observation: `Your top 10 conversation partners are mostly from ${yearsAgo}+ years ago. Your "active" network might be getting a bit dated?`,
                prompt: `Who have you met recently that could become a go-to connection? Sometimes we forget to nurture new relationships.`,
                action: 'Message someone new'
            });
        }
    }

    // 2. Warm Intros Available
    if (strategic.length > 0) {
        const warmCount = strategic.filter(c => c.isWarm).length;
        const coldCount = strategic.length - warmCount;

        insights.push({
            id: 'warm-intros',
            type: 'opportunity',
            icon: '🤝',
            headline: `You have ${strategic.length} connections at target companies`,
            observation: `${warmCount} of them are "warm" (you've messaged before), ${coldCount} are cold. These are your fastest path to referrals.`,
            prompt: `Have you reached out to the warm ones recently? A "hey, I'm exploring opportunities" message is totally fair game.`,
            action: 'Reach out for referrals'
        });
    }

    // 3. Network Composition
    if (founders.length > 50) {
        insights.push({
            id: 'founder-heavy',
            type: 'composition',
            icon: '🚀',
            headline: `You know ${founders.length} founders and CEOs`,
            observation: `That's a solid founder network! These connections are gold for startup opportunities, angel investing intros, or just learning what's being built.`,
            prompt: `Are you tapping into this? Founders love helping other founders (and aspiring ones).`,
            action: 'Engage your founder network'
        });
    }

    if (investors.length > 20) {
        insights.push({
            id: 'investor-access',
            type: 'composition',
            icon: '💰',
            headline: `You have ${investors.length} investors/VCs in your network`,
            observation: `That's real access to capital if you ever need it. Even if you're not fundraising, these folks see tons of companies and trends.`,
            prompt: `When's the last time you grabbed coffee with one? They often know about stealth roles and portfolio company needs.`,
            action: 'Reconnect with investors'
        });
    }

    if (recruiters.length > 10) {
        insights.push({
            id: 'recruiter-network',
            type: 'composition',
            icon: '🎯',
            headline: `You're connected to ${recruiters.length} recruiters`,
            observation: `Nice! Recruiters are most useful when you stay top of mind - not just when you're actively looking.`,
            prompt: `Do they know what you're looking for? A quick "here's what I'd move for" message keeps you on their radar.`,
            action: 'Update your recruiters'
        });
    }

    // 4. Dormant VIPs
    const seniorDormant = dormant.filter(c => {
        const t = (c.position || '').toLowerCase();
        return t.includes('ceo') || t.includes('founder') || t.includes('chief') ||
               t.includes('vp') || t.includes('director') || t.includes('partner');
    });

    if (seniorDormant.length > 20) {
        insights.push({
            id: 'dormant-vips',
            type: 'dormant',
            icon: '💎',
            headline: `${seniorDormant.length} senior people you haven't talked to in a year+`,
            observation: `Directors, VPs, founders... people who could open doors. Connections fade if you don't maintain them.`,
            prompt: `Pick 3-5 and send a genuine "thinking of you" or "saw your company in the news" message. No ask, just warmth.`,
            action: 'Warm up cold VIPs'
        });
    }

    // 5. Response Patterns (simplified - based on message counts)
    const oneWayConversations = partners.filter(p => p.messageCount === 1).length;
    const deepConversations = partners.filter(p => p.messageCount >= 10).length;

    if (oneWayConversations > deepConversations * 3 && oneWayConversations > 50) {
        insights.push({
            id: 'shallow-convos',
            type: 'pattern',
            icon: '💬',
            headline: `Lots of one-message conversations`,
            observation: `You have ${oneWayConversations} connections where you've only exchanged 1 message. Compare that to ${deepConversations} deeper conversations (10+ messages).`,
            prompt: `Quality > quantity in networking. A few deep relationships beat hundreds of "nice to meet you" messages.`,
            action: 'Deepen key relationships'
        });
    }

    return insights;
}
```

---

## Task 4: Create InsightCard Component

**Files:** `linkedin-insights.html` (add after RoleConnections)

```jsx
function InsightCard({ insight }) {
    return (
        <div className="bg-gray-800 rounded-xl p-6 shadow-lg border border-gray-700 hover:border-gray-600 transition">
            <div className="flex items-start gap-4">
                <div className="text-3xl">{insight.icon}</div>
                <div className="flex-1">
                    <h3 className="text-lg font-semibold text-white mb-2">{insight.headline}</h3>
                    <p className="text-gray-300 mb-3">{insight.observation}</p>
                    <p className="text-linkedin italic mb-4">{insight.prompt}</p>
                    <button className="px-4 py-2 bg-linkedin/20 text-linkedin rounded-lg text-sm font-medium hover:bg-linkedin/30 transition">
                        {insight.action} →
                    </button>
                </div>
            </div>
        </div>
    );
}
```

---

## Task 5: Create CareerActions Component

**Files:** `linkedin-insights.html` (add new component)

```jsx
function CareerActions({ insights, careerInsights }) {
    if (!careerInsights || careerInsights.length === 0) {
        return (
            <div className="space-y-6">
                <h2 className="text-2xl font-bold flex items-center gap-2">
                    <span>🎯</span> Career Actions
                </h2>
                <div className="bg-gray-800 rounded-xl p-8 text-center border border-gray-700">
                    <p className="text-gray-400">No specific insights right now - your career trajectory looks pretty standard!</p>
                </div>
            </div>
        );
    }

    return (
        <div className="space-y-6">
            <h2 className="text-2xl font-bold flex items-center gap-2">
                <span>🎯</span> Career Actions
            </h2>
            <p className="text-gray-400">Observations from your career history - things an interviewer or mentor might notice.</p>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                {careerInsights.map(insight => (
                    <InsightCard key={insight.id} insight={insight} />
                ))}
            </div>
        </div>
    );
}
```

---

## Task 6: Create RelationshipIntel Component

**Files:** `linkedin-insights.html` (add new component)

```jsx
function RelationshipIntel({ insights, relationshipInsights }) {
    if (!relationshipInsights || relationshipInsights.length === 0) {
        return (
            <div className="space-y-6">
                <h2 className="text-2xl font-bold flex items-center gap-2">
                    <span>💬</span> Relationship Intel
                </h2>
                <div className="bg-gray-800 rounded-xl p-8 text-center border border-gray-700">
                    <p className="text-gray-400">No specific insights right now. Your network looks healthy!</p>
                </div>
            </div>
        );
    }

    return (
        <div className="space-y-6">
            <h2 className="text-2xl font-bold flex items-center gap-2">
                <span>💬</span> Relationship Intel
            </h2>
            <p className="text-gray-400">Patterns in how you network - and opportunities you might be missing.</p>

            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                {relationshipInsights.map(insight => (
                    <InsightCard key={insight.id} insight={insight} />
                ))}
            </div>
        </div>
    );
}
```

---

## Task 7: Wire Up Dashboard

**Files:** `linkedin-insights.html` (Dashboard component)

**Step 1:** Update useEffect to compute new insights:
```jsx
const calculated = {
    // ... existing ...
    careerInsights: DataUtils.getCareerInsights(
        data.positions || [],
        data.jobApplications || [],
        data.savedJobs || []
    ),
    relationshipInsights: DataUtils.getRelationshipInsights(
        data,
        messagePartners,
        dormant,
        strategic,
        DataUtils.getConnectionsByRole(data.connections || [], 'founders'),
        DataUtils.getConnectionsByRole(data.connections || [], 'recruiters'),
        DataUtils.getConnectionsByRole(data.connections || [], 'investors'),
        DataUtils.getConnectionsByRole(data.connections || [], 'executives')
    )
};
```

**Step 2:** Update tab navigation to show 4 tabs

**Step 3:** Update content rendering:
```jsx
{activeTab === 'network' && <NetworkHealth data={data} insights={insights} intent={intent} />}
{activeTab === 'careerActions' && <CareerActions insights={insights} careerInsights={insights.careerInsights} />}
{activeTab === 'relationshipIntel' && <RelationshipIntel insights={insights} relationshipInsights={insights.relationshipInsights} />}
{activeTab === 'career' && <CareerTrajectory data={data} insights={insights} />}
```

---

## Summary

| Task | What It Builds |
|------|----------------|
| 1 | Update tab navigation to 4 tabs |
| 2 | Career insights computation (tenure, job search, progression, patterns) |
| 3 | Relationship insights computation (recency, intros, composition, dormant VIPs, patterns) |
| 4 | InsightCard component with curious friend voice |
| 5 | CareerActions tab component |
| 6 | RelationshipIntel tab component |
| 7 | Wire up Dashboard with new tabs and insights |

**Voice Consistency:**
- Headlines state the observation neutrally
- Observations add context without judgment ("Not judging!", "Totally valid", "That's not bad if...")
- Prompts ask reflective questions ("Have you thought about...", "What's holding you back...")
- Actions are clear next steps
