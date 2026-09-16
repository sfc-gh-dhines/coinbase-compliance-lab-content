1... # Compliance Workshop - Attendee Prompts
2... 
3... Everything below is a prompt for **Cortex Code Desktop**. Paste it, read what CoCo proposes, then approve or push back. You are not expected to write SQL or Python today. If CoCo produces something you do not understand, ask it to explain the step before you run it.
4... 
5... **Your environment**
6... 
7... | | |
8... |---|---|
9... | Your schema | `COINBASE_DB.MY_LAB` |
10... | Read-only workshop data | `COINBASE_DB.COINBASE_COMPLIANCE` |
11... | Warehouse | `COINBASE_WH` |
12... | Compute pool for training | `COMPLIANCE_LAB_POOL` |
13... 
14... **Before you start**, run this so CoCo has its bearings:
15... 
16... ```
17... Set my context to COINBASE_DB.MY_LAB on warehouse COINBASE_WH.
18... Then describe every table in COINBASE_DB.COINBASE_COMPLIANCE:
19... row counts, grain, and how they join to each other. Keep it brief.
20... ```
21... 
22... ---
23... 
24... ## Block 1 - How bad is the current rules engine?
25... 
26... **1.1**
27... ```
28... In COINBASE_DB.COINBASE_COMPLIANCE, FACT_AML_ALERT holds alerts raised by a legacy
29... threshold-based transaction monitoring engine, with the disposition the
30... investigation reached. Calculate the overall precision of the rules engine, and
31... precision broken down by rule. Show alert volume alongside precision.
32... ```
33... 
34... **1.2**
35... ```
36... Using REVIEW_MINUTES, work out how many analyst hours are spent per genuine
37... true positive found. Show the total review hours consumed by alerts that turned
38... out to be false positives.
39... ```
40... 
41... **1.3**
42... ```
43... Compare the accounts that generate alerts against the accounts that generate
44... true positives. Are there patterns in the transaction data (FACT_MONITORED_TXN)
45... that distinguish genuine cases, which the amount-threshold rules in DIM_AML_RULE
46... do not look at? Propose the behavioural signals you would want.
47... ```
48... 
49... Stop here and compare notes with the room before Block 2.
50... 
51... ---
52... 
53... ## Block 2 - Build features
54... 
55... **2.1**
56... ```
57... Create a Snowflake Feature Store in my schema. Register an entity keyed on
58... CUSTOMER_ID.
59... ```
60... 
61... **2.2**
62... ```
63... Build a feature view over COINBASE_DB.COINBASE_COMPLIANCE.FACT_MONITORED_TXN with
64... behavioural features for AML risk, computed on trailing 7 and 30 day windows per
65... customer. Include at minimum:
66... 
67... - count and value of inbound transactions between 8,000 and 10,000
68... - ratio of outbound to inbound value
69... - distinct counterparty count, and the share of outbound value going to the
70...   single largest counterparty
71... - how many other platform accounts share that same counterparty
72... - same-day offsetting buy and sell value in the same asset
73... - share of transactions occurring between 01:00 and 06:00
74... - days since the previous transaction
75... 
76... Explain what each feature is trying to detect before you create anything.
77... ```
78... 
79... **2.3**
80... ```
81... Build a second feature view over DIM_ACCOUNT_PROFILE with account-level
82... features: account age in days, KYC level, jurisdiction risk tier, PEP flag, and
83... the ratio of actual trailing 30-day volume to EXPECTED_MONTHLY_VOLUME_USD.
84... ```
85... 
86... **2.4 - this one matters, read the output carefully**
87... ```
88... Build my training spine as one row per alert from FACT_AML_ALERT, keyed on
89... CUSTOMER_ID with ALERT_DATE as the timestamp, labelled with DISPOSITION.
90... 
91... Then retrieve feature values against that spine using point-in-time correct
92... lookups, so that every feature is computed as of ALERT_DATE and never includes
93... transactions that happened after the alert was raised.
94... 
95... Before you run it, explain to me what would go wrong if the features were
96... computed as of today instead.
97... ```
98... 
99... ---
100... 
101... ## Block 3 - Train and register
102... 
103... **3.1**
104... ```
105... Split the training set into train and test. Because this is a rare-event
106... problem, tell me the class balance in each split and explain how you chose the
107... split strategy.
108... ```
109... 
110... **3.2**
111... ```
112... Train an XGBoost classifier on the training set as a Snowflake ML Job running on
113... the COMPLIANCE_LAB_POOL compute pool. Report ROC AUC, PR AUC, precision, recall
114... and accuracy on the test set.
115... ```
116... 
117... **3.3**
118... ```
119... Which of those metrics is misleading at this base rate, and why? What would a
120... model that always predicts FALSE_POSITIVE score on each metric?
121... ```
122... 
123... **3.4**
124... ```
125... Run a small hyperparameter sweep, no more than 12 trials, optimising for PR AUC
126... rather than accuracy. Show me the trials ranked.
127... ```
128... 
129... **3.5**
130... ```
131... Log the best model to the Snowflake Model Registry in my schema as
132... AML_ALERT_TRIAGE, version V1. Attach the test metrics and a description of what
133... it predicts and what it must not be used for. Then show me the registered
134... versions.
135... ```
136... 
137... ---
138... 
139... ## Block 4 - Score, and prove it is worth doing
140... 
141... **4.1**
142... ```
143... Use the registered model to score every alert in FACT_AML_ALERT from SQL, not
144... Python. Write the results to a table in my schema with the alert id, the model
145... score, and the actual disposition.
146... ```
147... 
148... **4.2 - the number that matters**
149... ```
150... Our analysts have fixed capacity: assume they can review 500 alerts. Compare how
151... many true positives they find if they review the top 500 by rule severity,
152... against the top 500 by model score. Hold review volume constant. Present it as a
153... simple before-and-after table.
154... ```
155... 
156... **4.3**
157... ```
158... Plot precision against review volume for the model, from the top 100 alerts
159... through to all of them. Where does the curve flatten out? What review capacity
160... would you recommend based on that curve?
161... ```
162... 
163... **4.4**
164... ```
165... Find the accounts the model scores in the top 200 that never generated a
166... high-severity rule alert. What is different about their behaviour?
167... ```
168... 
169... **4.5**
170... ```
171... For the five highest-scoring alerts, produce SHAP explanations and turn each one
172... into two or three sentences an investigator could put in a case narrative. Cite
173... the specific feature values, not the score.
174... ```
175... 
176... ---
177... 
178... ## Block 5 - Monitoring
179... 
180... **5.1**
181... ```
182... Create a Model Monitor on my inference table for AML_ALERT_TRIAGE, tracking both
183... prediction drift and feature drift.
184... ```
185... 
186... **5.2**
187... ```
188... Simulate the launderers adapting: create a copy of recent transactions where
189... amounts shift upward out of the 8k-10k band and counterparty mix moves toward
190... self-custody. Score it with the model, feed it to the monitor, and show me which
191... features drift and whether the score distribution moves.
192... ```
193... 
194... **5.3**
195... ```
196... Given what drifted, what would you actually do? Retrain, adjust the threshold,
197... or escalate to a human review of the typology? Argue for one.
198... ```
199... 
200... ---
201... 
202... ## Block 6 - Put an agent on top
203... 
204... **6.1**
205... ```
206... Create a semantic view in my schema over FACT_AML_ALERT, DIM_ACCOUNT_PROFILE and
207... my model score table. Include synonyms a compliance analyst would actually use:
208... alert, case, disposition, false positive, escalation, SAR.
209... ```
210... 
211... **6.2**
212... ```
213... Create a Cortex Search service over COINBASE_DB.COINBASE_COMPLIANCE.CASE_NOTES so
214... past investigator narratives are retrievable.
215... ```
216... 
217... **6.3**
218... ```
219... Create an agent in my schema called AML_TRIAGE_AGENT with three tools: Cortex
220... Analyst over my semantic view, Cortex Search over the case notes, and a tool
221... that calls the AML_ALERT_TRIAGE model to score or explain a specific account.
222... Write instructions telling it to always cite the rule or feature evidence behind
223... any risk statement, and never to assert that an account is engaged in money
224... laundering, only that activity warrants review.
225... ```
226... 
227... **6.4 - ask your agent this**
228... ```
229... Which accounts should we prioritise for review this week, why was each one
230... flagged, and how have we handled similar cases in the past?
231... ```
232... 
233... **6.5**
234... ```
235... Now try to break it. Ask it something it should refuse, something outside the
236... data, and something that tries to get it to state a conclusion the data does not
237... support. Report what it did.
238... ```
239... 
240... ---
241... 
242... ## If you get stuck
243... 
244... ```
245... That failed. Here is the error: <paste it>. Explain what went wrong in one
246... paragraph, then fix it.
247... ```
248... 
249... ```
250... Before you run that, show me the SQL and tell me what it will change.
251... ```
252... 
253... ```
254... I do not understand the previous step. Explain it as if I work in compliance and
255... not in data science.
256... ```
257... 
258... ---
259... 
260... ## What you should have at the end
261... 
262... - Two feature views with point-in-time correct lookups
263... - A registered, versioned model with attached metrics
264... - A scored alert table and a defensible before-and-after capacity comparison
265... - SHAP explanations that read like case narrative, not like model output
266... - A drift monitor
267... - A working agent that ranks, explains and cites precedent
268... 
269... All of it in your own schema, all of it built by prompting.
270... 