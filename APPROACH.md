#sQuestions

1. Do we have FHIR or Epic api integeration as these softwares charge seperatly for api service
   (if api integration is available use api for getting information or update in application if not available screen based agent image based to get information update or create may be created as task for clinic employees )

2. the number states 2,800 calls is it per clinc or over all ?
   if over all architure designed to handle 3000 to 3500 calls if per clinic then design changes to handle 2800X14 = 39200 calls per day

3. How missed calls are handled are we dailing back how it's recorded and action taken verified ?

4. how long waiting customers handled hold with queue and average waiting time anouncment

5. summary and resolution of each call contact needed ?

6. 40% of them are on Medicare or a Medicare Advantage plan.

Biggest payer group by far, and it is not one flow. Need 2:

- Original Medicare (Parts A & B) - federal, no network, no referral, claims go to the MAC
- Medicare Advantage (Part C) - private plan, has network + referral + prior auth, benefits change per plan and per year

## Flow A - Original Medicare (Parts A & B)

Payer is Medicare. No network limits, no referrals, prior auth is rare (mostly DME / certain items). Patient pays 20% coinsurance unless a Medigap plan covers it.

agent steps:

1. verify member - name, DOB, Medicare number, Part A/B effective dates
2. confirm clinic accepts Medicare assignment / is taking new Medicare patients
3. capture secondary coverage - Medigap or Medicaid - billing needs to know who pays the 20%
4. schedule / route, no referral needed
5. drugs sit under a separate Part D plan - route drug questions to pharmacy or staff

escalate to staff: no secondary on a high cost service, Medicare + Medicaid (dual eligible), DME / prior auth items

## Flow B - Medicare Advantage (Part C)

Payer is a private plan, member card is not the red white blue card. Network, referrals and prior auth all apply.

agent steps:

1. capture plan name, plan ID, member ID, group, effective date
2. verify eligibility - 270/271 real time if we have API or portal, else capture and create task
3. check in network for this clinic - out of network = stop, route to staff
4. check referral required (usually PCP referral for specialist) and prior auth for the service
5. start prior auth, else create a task for clinic employees
6. schedule / route with the plan recorded on the appointment

escalate to staff: out of network, prior auth required, plan not in our payer list, member id mismatch, DME

## Both flows

- same fields written to EHR for both: payer, plan, member id, group, effective date, secondary payer - one data model, 2 rule sets
- annual enrollment is Oct 15 - Dec 7 and benefits change every year, so payer rules must be a config table, not hardcoded
- MA has CMS rules on what we can say (no steering / plan comparison) - scripts need compliance review per flow

Questions for the client:
a. the split - how much Original Medicare vs Medicare Advantage, and top MA plans by call volume
b. do we get real time eligibility (API / portal) or only capture info for staff
c. who owns prior auth and referral - agent or staff
d. top 10 payers by volume so the rules can be built and tested

7. what is average cost per call/chat a minute example ?
   To choose model llm asr tts and stt based of cost customer expect

#Asummtions

1.We assume phone numbers are saved and prefered language is saved so on incoming call based of the data agent language is configured (for cost cutting we may use small models which are good in english /spanish)
if db not available we choose small models with good accuracy in both english and spanish

2.All calls/chats are logged into system agent attended calls/chats resolution provided with it to
if needed human intervention if it was unable to connect to person (all useres busy) that moment it create task list to call later then later it checks user availibity call patient and connect with them so no calls are query goes unnoticed

3.Any patient related queries are not provided in SMS or call before verifying it if it's patient  
phone number check
DOB and any one additional personal question from db to verify before providing any phi info in call or chat

4.Walled LLM with BBA signed or self hosted llm

5.PHI information are not passed to llm directly

6.data stored in db encrypted like name,contact,dob user can delete their information any time via call or chat if they want

7.Agent always answer and info it's automated assistant and your call is recorded

8.Agents need to escalte transfer or inform patient to call 911 is code red agent should not answer themselves should always transfer or ask them to call 911

9. Agent allways summarize and get confirmation before doing any change operation

10. For any Appointment authroize and check eligibilty for creating appointment

Solution design

Serverless container based cloud architecture BBA signed from provider for scaling large volume of calls

Tech stack
Python
Fast API websocket
Celery Queues and jobs
Haystack/Langchain/custom Agent builder
Redis(session storage)
Postgres (Source of truth)
React (Portal)
Quadrant vector DB (Policy documents are ingested so can answer directly with reading entier file)
S3/same server to Store Audio still BBA signed
Telecom provider like twilio etc

Agents

Tools & Prompts setup are common for both chat and voice
so this makes setup reuseable agents for future purpose

Chat will be SSE so api can emit thinking processing events before actual answer
Pipeline simple
question sent

llm generate answer answer mean time tool call and thinking events are emitted to keep user intreactive

chat turns are limited not all turns sent

Query rewritten

after n number of turn summary generated for older chats

Citations Added for RAG based Q/A

Calls will be websocket based
each call first check if patient in db if yes
pick preferd language and talk in it

Pipline consist of
LLM Large Language model
TTS text to speech
SST speech to text

AUDIO processed in parllel websockt connected with VAD
when ever speech starts and stop like full stop sent to stt and added  
sent to llm mean time stil listens once user stop speaking
everything is processed together and sent to tts and answerd this way low latency is achived

all turns are stored in redis per session not every turn updated in postgres db to avoid increase in latency and minimize db hits instead we write batch updates

Agent Level

All actions logged with reason and citations to debug (dev only) for production minimized version not all log

Gaurd Rails Adedd for fetch data from DB or any API

DB access is not provided directly Instead API with restricted access to particular user in call

Architecture Diagram
[Read Documentation](./ARCHITECTURE.md)

Eight-week scope

1.setup system for one clinc
2.English and Spanish Read only operations no updates for now
3.Check and validate calls and accuracy on spanish
4.After Hours calls can be Done in next phase
As we will be focused on setuping up agent architecture chat and call pipeline first
Benchmark spanish accuracy
Testing and 2 to 3 iteration to get it to better shape
