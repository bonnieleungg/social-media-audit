<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>InstaStoryline Generator — Dashboard</title>

  <!-- Chart.js -->
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
    .search-row{display:flex;gap:12px;align-items:center;margin-top:16px;flex-wrap:wrap}
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
  <div id="root" class="app">
    <!-- content will be injected by JS -->
  </div>

<script>
/* ---------- Simple standalone app (no React) ---------- */
(function(){
  // Utilities
  function el(tag, attrs={}, children=[]){
    const node = document.createElement(tag);
    for(const k in attrs){
      if(k === 'class') node.className = attrs[k];
      else if(k === 'html') node.innerHTML = attrs[k];
      else node.setAttribute(k, attrs[k]);
    }
    (Array.isArray(children) ? children : [children]).forEach(c => {
      if(c == null) return;
      if(typeof c === 'string') node.appendChild(document.createTextNode(c));
      else node.appendChild(c);
    });
    return node;
  }
  function fmtTS(ts) {
    if(!ts) return '--';
    const d = new Date(ts);
    return d.toLocaleString('en-US', {timeZone: 'America/Los_Angeles'});
  }

  // Mock data & functions (same behavior as before)
  function randomInt(max){ return Math.floor(Math.random()*max); }
  async function mockFetchAccountData({handle, postsLimit=50, days=90, forceFresh=false}) {
    await new Promise(res => setTimeout(res, 400 + Math.random()*600));
    if(Math.random() < 0.03 && !forceFresh){
      const err = new Error('RateLimited'); err.code = 429; throw err;
    }
    const now = Date.now();
    const posts = [];
    const sampleCount = Math.min(postsLimit, 40 + Math.floor(Math.random()*30));
    for(let i=0;i<sampleCount;i++){
      const ts = new Date(now - (i * (1000*60*60*24) * (Math.random()*3+0.5)));
      const likes = Math.floor(Math.random()*1500);
      const comments = Math.floor(Math.random()*200);
      posts.push({
        id: `post_${i}`,
        timestamp: ts.toISOString(),
        caption: `Sample caption ${i} — product shot ${i%3===0 ? 'UGC' : 'detail' }`,
        media_type: i%5===0 ? 'carousel' : (i%4===0 ? 'video' : 'image'),
        images: [{thumbnail_url: `https://picsum.photos/seed/${encodeURIComponent(handle)}-${i}/400/700`}],
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
      profile_image: `https://picsum.photos/seed/profile-${encodeURIComponent(handle)}/120/120`,
      last_active: posts[0] ? posts[0].timestamp : new Date().toISOString()
    };
    return {profile, posts};
  }

  function mockGenerateStorylines({profile, posts, brandVoice='playful', contentGoals='awareness'}){
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

  function computeEngagementMetrics(posts){
    const timeseries = posts.slice().reverse().map(p => ({
      t: new Date(p.timestamp).getTime(), likes: p.likes||0, comments: p.comments||0, saves:p.saves||0, shares:p.shares||0, followers:p.follower_count||0
    }));
    const totalInteractions = posts.reduce((s,p)=>s + (p.likes||0) + (p.comments||0) + (p.saves||0) + (p.shares||0), 0);
    const topPosts = posts.slice().map(p => {
      const interactions = (p.likes||0) + (p.comments||0) + (p.saves||0) + (p.shares||0);
      const rate = p.follower_count ? interactions / p.follower_count : 0;
      return {...p, interactions, engagement_rate: rate};
    }).sort((a,b)=>b.engagement_rate - a.engagement_rate);
    return {timeseries, totalInteractions, topPosts};
  }

  // Exports
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
  async function exportPDF(profile, storylines){
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

  // Build DOM
  const root = document.getElementById('root');

  // Header
  const header = el('header', {}, [
    el('div', {}, [
      el('h1', {}, ['InstaStoryline — Creative + Analytics Dashboard']),
      el('div', {class:'small muted'}, ['Generate multi-slide storylines, suggested shots, and engagement insights from public Instagram data.'])
    ]),
    el('div', {class:'controls'}, [
      el('div', {class:'export-btns'}, [
        el('button', {id:'export-json'}, ['Export JSON']),
        el('button', {id:'export-csv'}, ['Export CSV']),
        el('button', {id:'export-pdf'}, ['Export PDF'])
      ])
    ])
  ]);

  // Search row
  const searchRow = el('div', {class:'search-row'}, []);
  const inputHandle = el('input', {class:'input', value:'@samplebrand', placeholder:'@brand or instagram.com/brand'});
  const selectVoice = el('select', {class:'input'}, [
    el('option', {value:'playful'}, ['playful']),
    el('option', {value:'luxury'}, ['luxury']),
    el('option', {value:'educational'}, ['educational']),
    el('option', {value:'conversational'}, ['conversational'])
  ]);
  const selectGoals = el('select', {class:'input'}, [
    el('option', {value:'awareness'}, ['awareness']),
    el('option', {value:'conversions'}, ['conversions']),
    el('option', {value:'follower_growth'}, ['follower growth'])
  ]);
  const inputNumPosts = el('input', {class:'input', type:'number', min:10, max:200, value:50, style:'width:110px'});
  const selectTimeframe = el('select', {class:'input', style:'width:120px'}, [
    el('option', {value:30}, ['30 days']),
    el('option', {value:90}, ['90 days']),
    el('option', {value:180}, ['180 days'])
  ]);
  const btnSearch = el('button', {}, ['Search']);
  const btnForce = el('button', {style:'background:#f97316'}, ['Force fresh']);
  const statusDiv = el('div', {class:'status'}, [
    el('div', {class:'small muted', id:'last-ref'}, ['Last refreshed: --']),
    el('div', {style:'width:12px'}),
    el('div', {class:'dot success', id:'status-dot'})
  ]);

  searchRow.appendChild(inputHandle);
  searchRow.appendChild(selectVoice);
  searchRow.appendChild(selectGoals);
  searchRow.appendChild(inputNumPosts);
  searchRow.appendChild(selectTimeframe);
  const searchBtns = el('div', {style:'display:flex;gap:8px'}, [btnSearch, btnForce]);
  searchRow.appendChild(searchBtns);
  searchRow.appendChild(statusDiv);

  // Main grid
  const grid = el('div', {class:'grid'}, []);
  const mainCol = el('div', {}, []);
  const aside = el('aside', {}, []);

  // Main profile card
  const profileCard = el('div', {class:'card'}, []);
  mainCol.appendChild(profileCard);

  // Top posts card
  const topPostsCard = el('div', {class:'card', style:'margin-top:12px'}, []);
  mainCol.appendChild(topPostsCard);

  // Storylines card
  const storyCard = el('div', {class:'card', style:'margin-top:12px'}, []);
  mainCol.appendChild(storyCard);

  // Aside quick actions
  const quickCard = el('div', {class:'card'}, []);
  aside.appendChild(quickCard);

  const snapshotCard = el('div', {class:'card', style:'margin-top:12px'}, []);
  aside.appendChild(snapshotCard);

  const galleryCard = el('div', {class:'card', style:'margin-top:12px'}, []);
  aside.appendChild(galleryCard);

  const recCard = el('div', {class:'card', style:'margin-top:12px'}, []);
  aside.appendChild(recCard);

  grid.appendChild(mainCol);
  grid.appendChild(aside);

  root.appendChild(header);
  root.appendChild(searchRow);
  root.appendChild(grid);
  root.appendChild(el('div', {class:'footer'}, [el('div', {}, ['Acceptance: For accounts with ≥10 posts, the tool returns 5 storylines, engagement graphs, and top posts list. Use "Force fresh" to bypass cache.'])]))


  // Populate profile card UI (skeleton)
  function renderProfile(profile, analytics){
    profileCard.innerHTML = '';
    const headerDiv = el('div', {style:'display:flex;justify-content:space-between;align-items:center'}, [
      el('div', {style:'display:flex;gap:12px;align-items:center'}, [
        el('img', {src: profile.profile_image, style:'width:64px;height:64px;border-radius:12px;object-fit:cover'}),
        el('div', {}, [
          el('div', {style:'font-size:16px'}, [ profile.name + ' ', el('span', {class:'muted small'}, ['(' + profile.handle + ')']) ]),
          el('div', {class:'small muted'}, [profile.bio])
        ])
      ]),
      el('div', {style:'text-align:right'}, [
        el('div', {style:'font-size:18px;font-weight:600'}, [ (profile.follower_count || '--').toLocaleString ? profile.follower_count.toLocaleString() : profile.follower_count ]),
        el('div', {class:'small muted'}, ['followers'])
      ])
    ]);
    profileCard.appendChild(headerDiv);

    // top metrics and charts
    const topCards = el('div', {class:'top-cards', style:'margin-top:12px'}, [
      el('div', {class:'metric'}, [
        el('div', {class:'small muted'}, ['Engagement (period)']),
        el('div', {style:'font-size:18px;font-weight:600'}, [analytics.totalInteractions?.toLocaleString() || 0]),
        el('div', {class:'small muted'}, ['Total Likes+Comments+Saves+Shares'])
      ]),
      el('div', {class:'metric'}, [
        el('div', {class:'small muted'}, ['Top posts analyzed']),
        el('div', {style:'font-size:18px;font-weight:600'}, [analytics.topPosts?.length || '--']),
        el('div', {class:'small muted'}, ['Top posts by engagement rate'])
      ])
    ]);
    profileCard.appendChild(topCards);

    // charts container
    const charts = el('div', {style:'display:flex;gap:12px;margin-top:12px'}, [
      el('div', {style:'flex:1'}, [ el('div', {style:'height:180px'}, [ el('canvas', {id:'likesChart', style:'width:100%;height:100%'}) ]) ]),
      el('div', {style:'width:220px'}, [ el('div', {style:'height:180px'}, [ el('canvas', {id:'followersChart', style:'width:100%;height:100%'}) ]) ])
    ]);
    profileCard.appendChild(charts);

    // data source + cache button
    const bottom = el('div', {style:'display:flex;justify-content:space-between;align-items:center;margin-top:12px'}, [
      el('div', {class:'small muted'}, ['Data source status: ', el('strong', {class:'small'}, [document.getElementById('status-dot').dataset.msg || 'OK'])]),
      el('div', {}, [ el('button', {id:'clear-cache'}, ['Clear cache & refresh']) ])
    ]);
    profileCard.appendChild(bottom);

    // notice
    profileCard.appendChild(el('div', {class:'notice'}, ['This tool uses publicly available profile data. We do not access private DMs or private posts. ', el('a', {href:'#', style:'color:#9bbcff'}, ['Privacy policy']) ]));

    // Render charts
    try {
      const ts = analytics.timeseries || [];
      const likesCtx = document.getElementById('likesChart').getContext('2d');
      if(window._likesChart) window._likesChart.destroy();
      window._likesChart = new Chart(likesCtx, {
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

      const followersCtx = document.getElementById('followersChart').getContext('2d');
      if(window._followersChart) window._followersChart.destroy();
      window._followersChart = new Chart(followersCtx, {
        type:'line',
        data: {
          labels: ts.map(p => new Date(p.t)),
          datasets:[{label:'Followers', data: ts.map(p=>p.followers), borderWidth:2, fill:false, tension:0.2}]
        },
        options: {scales:{x:{type:'time', time:{unit:'day'}}}, plugins:{legend:{labels:{color:'#cbd5e1'}}}, maintainAspectRatio:false}
      });
    } catch(e) {
      console.error('Chart render error', e);
    }
  }

  function renderTopPostsCard(analytics){
    topPostsCard.innerHTML = '';
    topPostsCard.appendChild(el('div', {}, [ el('strong', {}, ['Top posts by engagement rate']), el('span', {class:'small muted', style:'margin-left:8px'}, ['(' + (analytics.topPosts?.length || 0) + ')']) ]));
    const tableWrap = el('div', {style:'margin-top:8px'});
    const table = el('table', {});
    const thead = el('thead', {}, [ el('tr', {}, [el('th', {}, ['Thumb']), el('th', {}, ['Caption']), el('th', {}, ['Interactions']), el('th', {}, ['Eng. rate'])]) ]);
    const tbody = el('tbody', {});
    (analytics.topPosts || []).slice(0,10).forEach(p => {
      const tr = el('tr', {}, [
        el('td', {}, [ el('img', {src: p.images?.[0]?.thumbnail_url, style:'width:56px;height:86px;object-fit:cover;border-radius:6px'}) ]),
        el('td', {style:'max-width:320px'}, [ (p.caption||'').slice(0,80) ]),
        el('td', {}, [ (p.interactions||0).toLocaleString() ]),
        el('td', {}, [ ((p.engagement_rate||0)*100).toFixed(2) + '%' ])
      ]);
      tbody.appendChild(tr);
    });
    table.appendChild(thead);
    table.appendChild(tbody);
    tableWrap.appendChild(table);
    topPostsCard.appendChild(tableWrap);
  }

  function renderStorylines(storylines){
    storyCard.innerHTML = '';
    storyCard.appendChild(el('div', {}, [ el('strong', {}, ['Storyline generator']), el('div', {class:'small muted'}, ['Auto-generated storylines tailored to brand voice & top exemplars']) ]));
    const container = el('div', {style:'margin-top:10px'});
    storylines.forEach(s => {
      const sdiv = el('div', {class:'storyline'}, [
        el('div', {style:'display:flex;justify-content:space-between;align-items:center'}, [
          el('div', {style:'font-weight:600'}, [s.title]),
          el('div', {class:'small muted'}, ['KPI: ' + s.kpi])
        ])
      ]);
      const inner = el('div', {style:'display:flex;gap:10px;margin-top:8px'}, [
        el('div', {style:'flex:1'}, s.slides.map(sl => el('div', {class:'slide'}, [ el('strong', {}, ['Slide ' + sl.slide_index]), ' — ' + sl.visual_direction, el('div', {class:'small muted'}, [sl.caption_text + ' • CTA: ' + sl.cta]) ]))),
        el('div', {style:'width:160px'}, [ el('div', {class:'gallery'}, s.sample_thumbnails.map(t => el('img', {src:t, class:'thumb'})) ) ])
      ]);
      sdiv.appendChild(inner);
      const details = el('details', {style:'margin-top:8px'}, [ el('summary', {class:'small muted'}, ['Suggested shots & prompts']), el('ul', {}, s.suggested_shots.map(sh => el('li', {class:'small muted'}, [`${sh.orientation} — ${sh.framing}; props: ${sh.props}; palette: ${sh.palette}`]))) ]);
      sdiv.appendChild(details);
      container.appendChild(sdiv);
    });
    storyCard.appendChild(container);
  }

  function renderAside(profile, analytics){
    quickCard.innerHTML = '';
    quickCard.appendChild(el('div', {style:'display:flex;justify-content:space-between;align-items:center'}, [ el('div', {}, [el('strong', {}, ['Quick actions'])]), el('div', {class:'small muted'}, ['Cached']) ]));
    const actions = el('div', {style:'margin-top:10px;display:flex;flex-direction:column;gap:8px'}, [
      el('button', {id:'export-json-2'}, ['Download JSON']),
      el('button', {id:'export-csv-2'}, ['Download CSV']),
      el('button', {id:'export-pdf-2'}, ['Download PDF brief']),
      el('button', {id:'clear-all-cache'}, ['Clear local cache'])
    ]);
    quickCard.appendChild(actions);

    snapshotCard.innerHTML = '';
    snapshotCard.appendChild(el('div', {}, [el('strong', {}, ['Profile snapshot'])]));
    snapshotCard.appendChild(el('div', {class:'small muted', style:'margin-top:8px'}, [
      el('div', {}, ['Handle: ', el('strong', {}, [profile?.handle || '--']) ]),
      el('div', {}, ['Last active: ', profile?.last_active ? new Date(profile.last_active).toLocaleDateString() : '--']),
      el('div', {}, ['Last fetch: ', document.getElementById('last-ref').dataset.ts ? fmtTS(+document.getElementById('last-ref').dataset.ts) : '--'])
    ]));

    galleryCard.innerHTML = '';
    galleryCard.appendChild(el('div', {}, [el('strong', {}, ['Top posts gallery'])]));
    const g = el('div', {class:'gallery', style:'margin-top:8px'});
    (analytics.topPosts || []).slice(0,8).forEach(p => g.appendChild(el('img', {src: p.images?.[0]?.thumbnail_url, class:'thumb'})));
    galleryCard.appendChild(g);

    recCard.innerHTML = '';
    recCard.appendChild(el('div', {}, [el('strong', {}, ['Recommendations (automated)'])]));
    recCard.appendChild(el('ul', {style:'margin-top:8px;font-size:13px;color:#cbd5e1'}, [
      el('li', {}, ['Increase carousel posts — correlated with higher engagement in top quartile.']),
      el('li', {}, ['Post 1–2x per week at historically high-engagement windows (see analytics).']),
      el('li', {}, ['Use more close-ups and faces in product shots.']),
      el('li', {}, ['Test polls + swipe CTA in stories to boost replies.'])
    ]));
  }

  // Main state
  let STATE = {profile:null, posts:[], analytics: {timeseries:[], totalInteractions:0, topPosts:[]}, storylines:[]};

  async function performSearch({force=false} = {}){
    const q = inputHandle.value.trim();
    const resolved = resolveHandleInput(q);
    if(!resolved){ alert('Enter an Instagram handle or public URL'); return; }
    setStatus('loading', 'Fetching account data...');
    const cacheKey = `insta_cache:${resolved}:${selectTimeframe.value}:${inputNumPosts.value}`;
    try {
      let fetched;
      if(!force){
        const cached = getCache(cacheKey);
        if(cached){ fetched = cached; setStatus('success','Loaded from cache'); }
      }
      if(!fetched){
        const payload = await mockFetchAccountData({handle: resolved, postsLimit: Number(inputNumPosts.value), days: Number(selectTimeframe.value), forceFresh: force});
        fetched = payload;
        setCache(cacheKey, fetched);
        setStatus('success','Fetched latest public posts');
      }
      STATE.profile = fetched.profile;
      STATE.posts = fetched.posts;
      document.getElementById('last-ref').textContent = 'Last refreshed: ' + fmtTS(Date.now());
      document.getElementById('last-ref').dataset.ts = Date.now();
      document.getElementById('last-ref').dataset.msg = 'OK';
      setStatusDotClass('success');
      STATE.analytics = computeEngagementMetrics(STATE.posts);
      STATE.storylines = mockGenerateStorylines({profile: STATE.profile, posts: STATE.posts, brandVoice: selectVoice.value, contentGoals: selectGoals.value});
      // render everything
      renderProfile(STATE.profile, STATE.analytics);
      renderTopPostsCard(STATE.analytics);
      renderStorylines(STATE.storylines);
      renderAside(STATE.profile, STATE.analytics);
    } catch(err){
      if(err && err.code === 429){ setStatus('rate', 'Rate limited — retry later'); setStatusDotClass('rate'); alert('Rate limited by source. Try again in a few minutes.'); }
      else { setStatus('error', 'Error: ' + (err.message||err)); setStatusDotClass('error'); console.error(err); alert('Error fetching data. See console.'); }
    }
  }

  // Helpers: simple cache
  const CACHE_TTL_MS = 15 * 60 * 1000;
  function setCache(key, value){ localStorage.setItem(key, JSON.stringify({ts:Date.now(), value})); }
  function getCache(key){
    try {
      const raw = localStorage.getItem(key); if(!raw) return null;
      const parsed = JSON.parse(raw); if(Date.now() - parsed.ts > CACHE_TTL_MS){ localStorage.removeItem(key); return null; }
      return parsed.value;
    } catch(e){ return null; }
  }
  function clearCacheKey(key){ localStorage.removeItem(key); }

  function setStatus(state, msg){
    document.getElementById('status-dot').dataset.msg = msg || state;
    document.getElementById('last-ref').textContent = 'Last refreshed: --';
  }
  function setStatusDotClass(cls){
    const dot = document.getElementById('status-dot');
    dot.className = 'dot ' + (cls==='success' ? 'success' : (cls==='rate' ? 'rate' : 'error'));
    dot.title = dot.dataset.msg || '';
    document.getElementById('last-ref').textContent = 'Last refreshed: ' + (document.getElementById('last-ref').dataset.ts ? fmtTS(+document.getElementById('last-ref').dataset.ts) : '--');
  }

  function resolveHandleInput(q){
    if(!q) return null;
    const urlMatch = q.match(/instagram\.com\/([A-Za-z0-9._]+)/i);
    if(urlMatch) return '@' + urlMatch[1];
    const atMatch = q.match(/^@?([A-Za-z0-9._]+)$/);
    if(atMatch) return '@' + atMatch[1];
    return null;
  }

  // Wire up events
  btnSearch.addEventListener('click', ()=>performSearch({force:false}));
  btnForce.addEventListener('click', ()=>performSearch({force:true}));
  document.addEventListener('click', function(e){
    if(e.target && e.target.id === 'clear-cache'){ 
      const resolved = resolveHandleInput(inputHandle.value.trim());
      if(resolved) clearCacheKey(`insta_cache:${resolved}:${selectTimeframe.value}:${inputNumPosts.value}`);
      performSearch({force:true});
    }
    if(e.target && e.target.id === 'clear-all-cache'){ if(confirm('Clear all cached accounts?')){ localStorage.clear(); alert('Cache cleared'); } }
    if(e.target && e.target.id === 'export-json'){ if(!STATE.profile) return alert('No data'); downloadJSON(`analysis-${STATE.profile.handle}.json`, {profile:STATE.profile, posts:STATE.posts, analytics:STATE.analytics, storylines:STATE.storylines}); }
    if(e.target && e.target.id === 'export-json-2'){ if(!STATE.profile) return alert('No data'); downloadJSON(`analysis-${STATE.profile.handle}.json`, {profile:STATE.profile, posts:STATE.posts, analytics:STATE.analytics, storylines:STATE.storylines}); }
    if(e.target && e.target.id === 'export-csv'){ if(!STATE.posts.length) return alert('No posts'); downloadCSV(`posts-${STATE.profile?.handle||'data'}.csv`, STATE.posts); }
    if(e.target && e.target.id === 'export-csv-2'){ if(!STATE.posts.length) return alert('No posts'); downloadCSV(`posts-${STATE.profile?.handle||'data'}.csv`, STATE.posts); }
    if(e.target && e.target.id === 'export-pdf'){ if(!STATE.profile) return alert('No profile loaded'); exportPDF(STATE.profile, STATE.storylines); }
    if(e.target && e.target.id === 'export-pdf-2'){ if(!STATE.profile) return alert('No profile loaded'); exportPDF(STATE.profile, STATE.storylines); }
  });

  // initial render (empty)
  renderProfile({profile_image:'https://picsum.photos/seed/profile-placeholder/120/120', name:'—', handle:'—', bio:'Profile bio will show here.', follower_count: '--'}, {timeseries:[], totalInteractions:0, topPosts:[]});
  renderTopPostsCard({topPosts:[]});
  renderStorylines([]);
  renderAside(null, {topPosts:[]});

  // run demo fetch on load
  performSearch({force:false}).catch(e => { console.error(e); });

})();
</script>
</body>
</html>
