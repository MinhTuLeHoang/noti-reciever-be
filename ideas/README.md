Use gh to create new repo noti-reciever-be

## Context: 

- I have many sources of notifications like: microsoft teams, microsoft outlook, my jenkin piplines (I have many repos, so there are many pipelines), gitlab ci pipelines, github action pipelines, cursor IDE noti when agent done, antigravity IDE noti when agent done.
- Currently it is difficult for me to distinct sound from all of them, some dont have sound too

## My idea:

- Create BE repo, public api. Then apply
    - Microsoft teams and outlooks through microsoft automations
    - jenkin, gitlab ci, github action through code
    - cursor IDE and antigravity IDE through IDE config (**Question 1**: Check possible here?)
- This api provide: 
    - source (enum)
    - message
- After intergate these source to call this api with privated token (fixed token), BE process to put into queue to send notifications to FE
- For FE, I will build in another repo. **Question 2**: should we implement window device and ios device seperate -> to improve performance of FE noti reciever from BE. If choosing coding 2 seperate platform, prioritize mac first.
- When FE recive notification from BE, it could speak the message from notification data (text to voice). It also have button to snooze (5min later), and understand button to mark it done and not noti again. (Thinking case, BE revice noti, and FE is not alive -> BE should store this noti and send it to FE when FE is alive).
- First of all, this is only internal use only, but implement so that in the fulture can be scalable to many users
- BE should have serve CRUD config, include:
    - source (enum noti source)
    - snooze time (string array: ["10s", "5m"])
    - level (enum: info, warning, error, success)
    - Action button (obj { method: enum, data: {} } ) this could be opening web, or calling webhook, or execute CMD command on FE (FE has list of allow cli)

## Open questions
Technically, about tech like:
- language and framework -> need you suggest
- deployment -> I need free service hosting (I dont have any domain, I accept free domain/subdomain too)
- For FE. which tech stack (language and framework should I apply too)
- Check for authen-author, if needed, I prefer free popuplar method: Google -> Facebook -> Github -> Microsoft (only implement 1 of them based on priority, dont self implement authen-author, prefer use 3rt party like I list)