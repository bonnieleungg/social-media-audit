#social-media-audit

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>InstaStoryline Generator — Dashboard</title>
  <!-- React + ReactDOM + Babel for JSX in the browser (dev convenience). For production, bundle with build tooling. -->
  <script crossorigin src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script crossorigin src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>

  <!-- Chart.js for charts -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <!-- jsPDF for PDF export -->
  <script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>

  <style>
    :root{
      --bg:#0f1724; --card:#0b1220; --accent:#60a5fa; --muted:#94a3b8; --glass: rgba(255,255,255,0.03);
      font-family: Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
    }
    body{margin:0;background:linear-gradient(180deg,#071028 0%, #071523 100%); color:#e6eef8;}
    .app{max-width:1200px;margin:28px auto;padding:20px;}
    header{display:flex;gap:20px;align-items:center;justify-content:space-between;}
    h1{font-size:20px;margin:0}
    .search-row{display:flex;gap:12px;align-items:center;margin-top:16px}
    .input, select{background:var(--glass);border:1px solid rgba(255,255,255,0.04);padding:10px 12px;border-radius:8px;color:var(--muted);min-width:200px}
    button{background:var(--accent);color:#04203a;border:none;padding:10px 12px;border-radius:8px;cursor:pointer}
    .grid{display:grid;grid-template-columns: 1fr 360px;gap:18px;margin-top:18px}
    .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));padding:14px;border-radius:12px;border:1px solid rgba(255,255,255,0.03)}
    .small{font-size:13px;color:var(--muted)}
    .muted{color:var(--muted)}
    .top-cards{display:flex;gap:12px}
    .metric{padding:10px;border-radius:10px;background:rgba(255,255,255,0.02);flex:1}
    .gallery{display:flex;flex-wrap:wrap;gap:8px}
    .thumb{width:72px;height:128px;border-radius:8px;object-fit:cover}
    .storyline{border:1px dashed rgba(255,255,255,0.03);padding:10px;border-radius:8px;margin-bottom:10px;background:linear-gradient(180deg, rgba(255,255,255,0.01), transparent)}
    .slide{padding:6px;border-radius:6px;background:rgba(255,255,255,0.015);margin-bottom:6px}
    table{width:100%;border-collapse:collapse;color:var(--muted);font-size:13px}
    th,td{padding:8px;border-bottom:1px solid rgba(255,255,255,0.02);text-align:left}
    .controls{display:flex;gap:8px;align-items:center}
    .status{display:flex;gap:10px;align-items:center}
    .dot{width:10px;height:10px;border-radius:50%}
    .dot.success{background:#34d399}
    .dot.error{background:#fb7185}
    .dot.rate{background:#f59e0b}
    .notice{font-size:12px;color:#cbd5e1;margin-top:10px}
    .footer{margin-top:18px;font-size:13px;color:var(--muted)}
    .export-btns{display:flex;gap:8px}
    @media (max-width:980px){.grid{grid-template-columns:1fr;}.search-row{flex-direction:column;align-items:flex-start}.top-cards{flex-direction:column}}
  </style>
</head>
<body>
  <div id="root"></div>

  <script type="text/babel">
    const {useState, useEffect, useRef} = React;

    /* ---------- UTIL: time formatting ---------- */
    function fmtTS(ts) {
      if(!ts) return '--';
      const d = new Date(ts);
      return d.toLocaleString('en-US', {timeZone: 'America/Los_Angeles'});
    }
    function nowISO(){ return (new Date()).toISOString(); }

    /* ---------- CACHE: short-lived cache (15min) ---------- */
    const CACHE_TTL_MS = 15 * 60 * 1000; // 15 minutes
    function setCache(key, value) {
      const payload = {ts: Date.now(), value};
      localStorage.setItem(key, JSON.stringify(payload));
    }
    function getCache(key) {
      try {
        const raw = localStorage.getItem(key);
        if(!raw) return null;
        const parsed = JSON.parse(raw);
        if(Date.now() - parsed.ts > CACHE_TTL_MS) {
          localStorage.removeItem(key);
          return null;
        }
        return parsed.value;
      } catch(e){ return null; }
    }
    function clearCache(key){ localStorage.removeItem(key); }

    /* ---------- MOCK: Replace these with real backend endpoints ---------- */

    // Mock fetcher: normally you'd call your scraping API: /api/fetch?handle=...
    // The returned object should match the Data model in your spec: Profile + Posts
    async function mockFetchAccountData({handle, postsLimit=50, days=90, forceFresh=false}) {
      // Simulate network latency and possible rate-limit errors
      await new Promise(res => setTimeout(res, 800 + Math.random()*800));
      // Simulate rate limit 5% of the time
      if(Math.random() < 0.05 && !forceFresh) {
        const err = new Error('RateLimited');
        err.code = 429;
        throw err;
      }
      // Minimal sample dataset generator
      const now = Date.now();
      const posts = [];
      const sampleCount = Math.min(postsLimit, 40 + Math.floor(Math.random()*30));
      for(let i=0;i<sampleCount;i++){
        const ts = new Date(now - (i * (1000*60*60*24) * (Math.random()*3+0.5))); // spread irregular
        const likes = Math.floor(Math.random()*1500);
        const comments = Math.floor(Math.random()*200);
        const engagement = likes + comments;
        posts.push({
          id: `post_${i}`,
          timestamp: ts.toISOString(),
          caption: `Sample caption ${i} — product shot ${i%3===0 ? 'UGC' : 'detail' }`,
          media_type: i%5===0 ? 'carousel' : (i%4===0 ? 'video' : 'image'),
          images: [{thumbnail_url: `https://picsum.photos/seed/${handle}-${i}/400/700`}],
          likes, comments,
          saves: Math.floor(Math.random()*200),
          shares: Math.floor(Math.random()*80),
          follower_count: 12000 + Math.floor(Math.random()*2000) - i*5
        });
      }
      const profile = {
        handle,
        name: handle.replace('@','').toUpperCase(),
        bio: "Auto-generated mock bio. Replace with real profile bio from API.",
        follower_count: posts[0] ? posts[0].follower_count : 10000,
        profile_image: `https://picsum.photos/seed/profile-${handle}/120/120`,
        last_active: posts[0] ? posts[0].timestamp : nowISO()
      };
      return {profile, posts};
    }

    // Mock storyline generator — replace with call to LLM / prompt template that includes brand voice & top exemplars
    async function mockGenerateStorylines({profile, posts, brandVoice='playful', contentGoals='awareness'}) {
      await new Promise(res => setTimeout(res, 700));
      // create 5 storylines
      const topStyle = posts.slice(0,6).map(p => p.images?.[0]?.thumbnail_url);
      const storylines = [];
      for(let s=0;s<5;s++){
        const slidesCount = 2 + (s % 4);
        const slides = [];
        for(let sl=0; sl<slidesCount; sl++){
          slides.push({
            slide_index: sl+1,
            visual_direction: sl===0 ? 'Hero product close-up, shallow depth' : (sl===1 ? 'Behind-the-scenes: hands-on demo' : 'UGC testimonial overlay with caption'),
            caption_text: sl===0 ? `${profile.name} — product moment` : 'Short supporting text (≤100 chars)',
            cta: sl===slidesCount-1 ? 'Swipe up / Link' : (sl%2===0 ? 'Poll' : 'DM')
          });
        }
        const suggested_shots = [];
        for(let r=0;r<3+ (s%3); r++){
          suggested_shots.push({
            orientation: 'vertical 9:16',
            framing: r===0 ? 'tight product close-up' : (r===1 ? '3/4 body with product' : 'overhead flat lay'),
            props: 'neutral props, minimal styling',
            palette: s%2===0 ? 'warm neutrals' : 'vibrant contrasting'
          });
        }
        storylines.push({
          id: `st_${s}`,
          title: `${brandVoice} Storyline #${s+1}`,
          slides,
          suggested_shots,
          kpi: s%2===0 ? 'increase story replies' : 'improve swipe-ups',
          sample_thumbnails: topStyle.slice(s, s+2)
        });
      }
      return storylines;
    }

    /* ---------- ANALYTICS: compute engagement rates, trends ---------- */
    function computeEngagementMetrics(posts) {
      // engagement_rate := (likes + comments) / follower_count
      const timeseries = posts.slice().reverse().map(p => {
        return {t: new Date(p.timestamp).getTime(), likes: p.likes||0, comments: p.comments||0, saves: p.saves||0, shares: p.shares||0, followers: p.follower_count || 0};
      });
      // Simple rolling aggregates for example
      const totalInteractions = posts.reduce((s,p)=>s + (p.likes||0) + (p.comments||0) + (p.saves||0) + (p.shares||0), 0);
      const topPosts = posts.slice().map(p => {
        const interactions = (p.likes||0) + (p.comments||0) + (p.saves||0) + (p.shares||0);
        const rate = p.follower_count ? interactions / p.follower_count : 0;
        return {...p, interactions, engagement_rate: rate};
      }).sort((a,b)=>b.engagement_rate - a.engagement_rate);
      return {timeseries, totalInteractions, topPosts};
    }

    /* ---------- CSV & JSON & PDF export helpers ---------- */
    function downloadJSON(filename, data){
      const blob = new Blob([JSON.stringify(data, null, 2)], {type:'application/json'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a'); a.href=url; a.download = filename; a.click(); URL.revokeObjectURL(url);
    }
    function downloadCSV(filename, posts) {
      const header = ['id','timestamp','caption','media_type','likes','comments','saves','shares','follower_count'];
      const rows = posts.map(p => header.map(h => JSON.stringify(p[h] ?? '')).join(','));
      const csv = [header.join(','), ...rows].join('\n');
      const blob = new Blob([csv], {type:'text/csv'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a'); a.href=url; a.download = filename; a.click(); URL.revokeObjectURL(url);
    }
    async function exportPDF({profile, storylines, analytics}) {
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF({unit:'pt', format:'a4'});
      doc.setFontSize(14);
      doc.text(`${profile.handle} — Storyline Brief`, 40, 50);
      doc.setFontSize(11);
      doc.text(`Generated: ${fmtTS(Date.now())}`, 40, 70);
      let y = 100;
      storylines.forEach((s, idx) => {
        doc.setFontSize(12);
        doc.text(`${idx+1}. ${s.title} — KPI: ${s.kpi}`, 40, y);
        y += 16;
        s.slides.forEach(sl => {
          doc.setFontSize(10);
          doc.text(`• Slide ${sl.slide_index}: ${sl.visual_direction} | ${sl.caption_text} | CTA: ${sl.cta}`, 60, y);
          y += 12;
        });
        y += 8;
        if(y > 700) { doc.addPage(); y = 40; }
      });
      doc.save(`storylines-${profile.handle.replace('@','')}.pdf`);
    }

    /* ---------- MAIN App ---------- */
    function App(){
      const [query, setQuery] = useState('@samplebrand');
      const [brandVoice, setBrandVoice] = useState('playful');
      const [contentGoals, setContentGoals] = useState('awareness');
      const [timeframeDays, setTimeframeDays] = useState(90);
      const [numPosts, setNumPosts] = useState(50);
      const [profileData, setProfileData] = useState(null);
      const [posts, setPosts] = useState([]);
      const [lastRefreshed, setLastRefreshed] = useState(null);
      const [status, setStatus] = useState({state:'idle', code:200, message:'No data'});
      const [storylines, setStorylines] = useState([]);
      const [analytics, setAnalytics] = useState({timeseries:[], totalInteractions:0, topPosts:[]});
      const [autoRefreshKey, setAutoRefreshKey] = useState(0);
      const [forceFresh, setForceFresh] = useState(false);
      const searchInProgress = status.state === 'loading';

      // Chart refs
      const likesChartRef = useRef(null);
      const followersChartRef = useRef(null);
      const likesChartInstance = useRef(null);
      const followersChartInstance = useRef(null);

      useEffect(() => {
        // when analytics.timeseries changes, update charts
        const ts = analytics.timeseries || [];
        // Likes/Comments timeseries chart
        if(likesChartRef.current){
          const ctx = likesChartRef.current.getContext('2d');
          if(likesChartInstance.current) likesChartInstance.current.destroy();
          likesChartInstance.current = new Chart(ctx, {
            type: 'line',
            data: {
              labels: ts.map(p => new Date(p.t)),
              datasets: [
                {label:'Likes', data: ts.map(p=>p.likes), borderWidth:2, fill:false, tension:0.2},
                {label:'Comments', data: ts.map(p=>p.comments), borderWidth:2, fill:false, tension:0.2}
              ]
            },
            options: {scales:{x:{type:'time', time:{unit:'day'}}}, plugins:{legend:{labels:{color:'#cbd5e1'}}}, maintainAspectRatio:false}
          });
        }
        // Followers chart
        if(followersChartRef.current){
          const ctx2 = followersChartRef.current.getContext('2d');
          if(followersChartInstance.current) followersChartInstance.current.destroy();
          followersChartInstance.current = new Chart(ctx2, {
            type: 'line',
            data: {
              labels: ts.map(p => new Date(p.t)),
              datasets: [{label:'Followers', data: ts.map(p=>p.followers), borderWidth:2, fill:false, tension:0.2}]
            },
            options: {scales:{x:{type:'time', time:{unit:'day'}}}, plugins:{legend:{labels:{color:'#cbd5e1'}}}, maintainAspectRatio:false}
          });
        }

      }, [analytics.timeseries]);

      /* ---------- CORE: perform search / fetch ---------- */
      async function performSearch({force=false} = {}) {
        const resolvedHandle = resolveHandleInput(query);
        if(!resolvedHandle) {
          setStatus({state:'error', code:400, message:'Provide an Instagram handle or public profile URL.'});
          return;
        }
        setStatus({state:'loading', code:0, message:'Fetching account data...'});
        const cacheKey = `insta_cache:${resolvedHandle}:${timeframeDays}:${numPosts}`;
        try {
          let fetched;
          if(!force){
            const cached = getCache(cacheKey);
            if(cached){ fetched = cached; setStatus({state:'success', code:200, message:'Loaded from cache'}); }
          }
          if(!fetched) {
            // call your real backend API here.
            // Example: const res = await fetch(`/api/fetch?handle=${resolvedHandle}&limit=${numPosts}&days=${timeframeDays}`);
            // const payload = await res.json();
            // We'll call mock for demo:
            const payload = await mockFetchAccountData({handle: resolvedHandle, postsLimit: numPosts, days: timeframeDays, forceFresh: force});
            fetched = payload;
            // store short-lived cache to avoid re-scraping repeatedly
            setCache(cacheKey, fetched);
            setStatus({state:'success', code:200, message:'Fetched latest public posts'});
          }
          setProfileData(fetched.profile);
          setPosts(fetched.posts);
          setLastRefreshed(Date.now());

          // run analytics
          const computed = computeEngagementMetrics(fetched.posts);
          setAnalytics(computed);

          // Generate storylines (LLM) - call your LLM endpoint with prompt template, brand voice, and top exemplars
          setStatus({state:'loading', code:0, message:'Generating storylines...'});
          // Replace mockGenerateStorylines with real API call to your LLM/analysis pipeline
          const generated = await mockGenerateStorylines({profile: fetched.profile, posts: fetched.posts, brandVoice, contentGoals});
          setStorylines(generated);
          setStatus({state:'success', code:200, message:'Analysis complete'});
        } catch(err) {
          if(err && err.code === 429) {
            setStatus({state:'rate_limited', code:429, message:'Rate limited — please retry after backoff window.'});
          } else {
            setStatus({state:'error', code: err?.code || 500, message: err?.message || 'Unknown error'});
          }
        }
      }

      /* ---------- HELPERS ---------- */
      function resolveHandleInput(q) {
        if(!q) return null;
        q = q.trim();
        // Accept formats: @handle, handle, full URL (instagram.com/handle)
        const urlMatch = q.match(/instagram\.com\/([A-Za-z0-9._]+)/i);
        if(urlMatch) return '@' + urlMatch[1];
        const atMatch = q.match(/^@?([A-Za-z0-9._]+)$/);
        if(atMatch) return '@' + atMatch[1];
        return null;
      }

      function forceSearch(){
        // Clear relevant cache and re-run fetch
        const resolved = resolveHandleInput(query);
        if(resolved){
          const cacheKey = `insta_cache:${resolved}:${timeframeDays}:${numPosts}`;
          clearCache(cacheKey);
        }
        performSearch({force:true});
      }

      function exportAllJSON(){ if(!profileData) return alert('No data to export'); downloadJSON(`analysis-${profileData.handle}.json`, {profile:profileData, posts, analytics, storylines}); }
      function exportCSV(){ if(!posts.length) return alert('No posts'); downloadCSV(`posts-${profileData?.handle || 'data'}.csv`, posts); }
      function exportPDF(){ if(!profileData) return alert('No profile loaded'); exportPDF({profile:profileData, storylines, analytics}); }

      // Auto-run a search on mount for demo
      useEffect(()=>{ performSearch(); }, []);


      /* ---------- Render ---------- */
      return (
        <div className="app">
          <header>
            <div>
              <h1>InstaStoryline — Creative + Analytics Dashboard</h1>
              <div className="small muted">Generate multi-slide storylines, suggested shots, and engagement insights from public Instagram data.</div>
            </div>
            <div className="controls">
              <div className="export-btns">
                <button onClick={exportAllJSON} title="Download JSON">Export JSON</button>
                <button onClick={exportCSV} title="Download CSV">Export CSV</button>
                <button onClick={exportPDF} title="Export PDF brief">Export PDF</button>
              </div>
            </div>
          </header>

          <div className="search-row">
            <input className="input" value={query} onChange={e=>setQuery(e.target.value)} placeholder="@brand or instagram.com/brand" />
            <select className="input" value={brandVoice} onChange={e=>setBrandVoice(e.target.value)}>
              <option value="playful">playful</option>
              <option value="luxury">luxury</option>
              <option value="educational">educational</option>
              <option value="conversational">conversational</option>
            </select>
            <select className="input" value={contentGoals} onChange={e=>setContentGoals(e.target.value)}>
              <option value="awareness">awareness</option>
              <option value="conversions">conversions</option>
              <option value="follower_growth">follower growth</option>
            </select>
            <input className="input" type="number" min="10" max="200" value={numPosts} onChange={e=>setNumPosts(Number(e.target.value))} style={{width:110}} />
            <select className="input" value={timeframeDays} onChange={e=>setTimeframeDays(Number(e.target.value))} style={{width:120}}>
              <option value={30}>30 days</option>
              <option value={90}>90 days</option>
              <option value={180}>180 days</option>
            </select>
            <div style={{display:'flex',gap:8}}>
              <button onClick={()=>performSearch({force:false})} disabled={searchInProgress}>{searchInProgress ? 'Working...' : 'Search'}</button>
              <button onClick={()=>{ setForceFresh(true); forceSearch(); }} style={{background:'#f97316'}}>Force fresh</button>
            </div>
            <div style={{marginLeft:'auto'}} className="status">
              <div className="small muted">Last refreshed: <strong>{lastRefreshed ? fmtTS(lastRefreshed) : '--'}</strong></div>
              <div style={{width:12}} />
              <div className="dot {status.state}"></div>
            </div>
          </div>

          <div className="grid">
            <div>
              <div className="card">
                <div style={{display:'flex',justifyContent:'space-between',alignItems:'center'}}>
                  <div style={{display:'flex',gap:12,alignItems:'center'}}>
                    <img src={profileData?.profile_image || 'https://picsum.photos/seed/profile-placeholder/80/80'} alt="pf" style={{width:64,height:64,borderRadius:12,objectFit:'cover'}}/>
                    <div>
                      <div style={{fontSize:16}}>{profileData?.name || '—' } <span className="muted small">({profileData?.handle||'—'})</span></div>
                      <div className="small muted">{profileData?.bio || 'Profile bio will show here.'}</div>
                    </div>
                  </div>
                  <div style={{textAlign:'right'}}>
                    <div style={{fontSize:18,fontWeight:600}}>{profileData?.follower_count?.toLocaleString() || '--'}</div>
                    <div className="small muted">followers</div>
                  </div>
                </div>

                <div style={{marginTop:12}} className="top-cards">
                  <div className="metric">
                    <div className="small muted">Engagement (period)</div>
                    <div style={{fontSize:18,fontWeight:600}}>{analytics.totalInteractions?.toLocaleString() || 0}</div>
                    <div className="small muted">Total Likes+Comments+Saves+Shares</div>
                  </div>
                  <div className="metric">
                    <div className="small muted">Top posts analyzed</div>
                    <div style={{fontSize:18,fontWeight:600}}>{analytics.topPosts?.length || posts.length}</div>
                    <div className="small muted">Top posts by engagement rate</div>
                  </div>
                </div>

                <div style={{display:'flex',gap:12,marginTop:12}}>
                  <div style={{flex:1}}>
                    <div style={{height:180}}><canvas ref={likesChartRef} style={{width:'100%',height:'100%'}}></canvas></div>
                  </div>
                  <div style={{width:220}}>
                    <div style={{height:180}}><canvas ref={followersChartRef} style={{width:'100%',height:'100%'}}></canvas></div>
                  </div>
                </div>

                <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginTop:12}}>
                  <div className="small muted">Data source status: <strong className="small">{status.message}</strong></div>
                  <div>
                    <button onClick={forceSearch}>Clear cache & refresh</button>
                  </div>
                </div>

                <div className="notice">
                  This tool uses publicly available profile data. We do not access private DMs or private posts. <a href="#" style={{color:'#9bbcff'}}>Privacy policy</a>.
                </div>
              </div>

              <div className="card" style={{marginTop:12}}>
                <div style={{display:'flex',justifyContent:'space-between',alignItems:'center'}}>
                  <div><strong>Top posts by engagement rate</strong> <span className="small muted">({analytics.topPosts?.length || 0})</span></div>
                </div>
                <div style={{marginTop:8}}>
                  <table>
                    <thead>
                      <tr><th>Thumb</th><th>Caption</th><th>Interactions</th><th>Eng. rate</th></tr>
                    </thead>
                    <tbody>
                      {(analytics.topPosts || []).slice(0,10).map(p => (
                        <tr key={p.id}>
                          <td><img src={p.images?.[0]?.thumbnail_url} alt="" style={{width:56,height:86,objectFit:'cover',borderRadius:6}}/></td>
                          <td style={{maxWidth:320}}>{p.caption?.slice(0,80)}</td>
                          <td>{p.interactions?.toLocaleString()}</td>
                          <td>{(p.engagement_rate * 100).toFixed(2)}%</td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
              </div>

              <div className="card" style={{marginTop:12}}>
                <div style={{display:'flex',justifyContent:'space-between',alignItems:'center'}}>
                  <div><strong>Storyline generator</strong> <div className="small muted">Auto-generated storylines tailored to brand voice & top exemplars</div></div>
                  <div className="small muted">Export as PDF / Brief</div>
                </div>
                <div style={{marginTop:10}}>
                  {storylines.map(s => (
                    <div className="storyline" key={s.id}>
                      <div style={{display:'flex',justifyContent:'space-between',alignItems:'center'}}>
                        <div style={{fontWeight:600}}>{s.title}</div>
                        <div className="small muted">KPI: {s.kpi}</div>
                      </div>
                      <div style={{display:'flex',gap:10,marginTop:8}}>
                        <div style={{flex:1}}>
                          {s.slides.map(sl => (
                            <div className="slide" key={sl.slide_index}>
                              <strong>Slide {sl.slide_index}</strong> — {sl.visual_direction} <div className="small muted">{sl.caption_text} • CTA: {sl.cta}</div>
                            </div>
                          ))}
                        </div>
                        <div style={{width:160}}>
                          <div className="gallery">
                            {s.sample_thumbnails?.map((t, i) => <img key={i} src={t} className="thumb" alt="sample"/>)}
                          </div>
                        </div>
                      </div>

                      <details style={{marginTop:8}}>
                        <summary className="small muted">Suggested shots & prompts</summary>
                        <ul>
                          {s.suggested_shots.map((sh, idx) => (
                            <li key={idx} className="small muted">{sh.orientation} — {sh.framing}; props: {sh.props}; palette: {sh.palette}</li>
                          ))}
                        </ul>
                      </details>
                    </div>
                  ))}
                </div>
              </div>

            </div>

            <aside>
              <div className="card">
                <div style={{display:'flex',justifyContent:'space-between',alignItems:'center'}}>
                  <div><strong>Quick actions</strong></div>
                  <div className="small muted">Cached</div>
                </div>
                <div style={{marginTop:10,display:'flex',flexDirection:'column',gap:8}}>
                  <button onClick={()=>exportAllJSON()}>Download JSON</button>
                  <button onClick={()=>exportCSV()}>Download CSV</button>
                  <button onClick={()=>exportPDF()}>Download PDF brief</button>
                  <button onClick={()=>{ if(confirm('Clear all cached accounts?')) localStorage.clear(); alert('Cache cleared'); }}>Clear local cache</button>
                </div>
              </div>

              <div className="card" style={{marginTop:12}}>
                <div><strong>Profile snapshot</strong></div>
                <div className="small muted" style={{marginTop:8}}>
                  <div>Handle: <strong>{profileData?.handle || '--'}</strong></div>
                  <div>Last active: {profileData?.last_active ? new Date(profileData.last_active).toLocaleDateString() : '--'}</div>
                  <div>Last fetch: {lastRefreshed ? fmtTS(lastRefreshed) : '--'}</div>
                </div>
              </div>

              <div className="card" style={{marginTop:12}}>
                <div><strong>Top posts gallery</strong></div>
                <div style={{marginTop:8}} className="gallery">
                  {(analytics.topPosts || []).slice(0,8).map(p=>(
                    <img key={p.id} src={p.images?.[0]?.thumbnail_url} className="thumb" alt="post" />
                  ))}
                </div>
              </div>

              <div className="card" style={{marginTop:12}}>
                <div><strong>Recommendations (automated)</strong></div>
                <ul style={{marginTop:8,fontSize:13,color:'#cbd5e1'}}>
                  <li>Increase carousel posts — correlated with higher engagement in top quartile.</li>
                  <li>Post 1–2x per week at historically high-engagement windows (see analytics).</li>
                  <li>Use more close-ups and faces in product shots.</li>
                  <li>Test polls + swipe CTA in stories to boost replies.</li>
                </ul>
              </div>

            </aside>
          </div>

          <div className="footer">
            <div>Acceptance: For accounts with ≥10 posts, the tool returns 5 storylines, engagement graphs, and top posts list. Use "Force fresh" to bypass cache. If rate-limited, the status will show a retry suggestion.</div>
          </div>
        </div>
      );
    }

    ReactDOM.createRoot(document.getElementById('root')).render(<App/>);
  </script>
</body>
</html>
