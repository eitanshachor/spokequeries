# spokequeries
An updating list for keeping Spoke queries.

### Proper Date Formatting
```SQL
to_char(u.created_at AT time zone 'mst', 'FMDy, FMMon FMDD, YYYY at FMHH:MI PM') as join_date
```
### Query I made w/Ricky that lists each campaign by campaign and responses
```SQL
SELECT
    campaign.id,
    title,
    interaction_step.question,
    q_value,
    answerz
FROM
    public.campaign
JOIN
    public.interaction_step ON campaign.id = interaction_step.campaign_id
JOIN
    (SELECT
        campaign_id AS camp_id,
        i_s.id AS i_s_id,
        q_r.value AS q_value,
        COUNT (q_r.value) AS answerz
    FROM 
        public.question_response AS q_r
    JOIN 
        public.interaction_step AS i_s ON q_r.interaction_step_id = i_s.id
    GROUP BY campaign_id, i_s.id, q_r.value
    ORDER BY campaign_id, i_s.id, answerz DESC) AS rickyz_table
ON campaign.id = camp_id AND interaction_step.id = i_s_id
```

### Texts Received All Time
```SQL
SELECT 
    COUNT (message.id) AS texts_sent_all_time
FROM
    public.message
JOIN
    campaign_contact ON campaign_contact.id = message.campaign_contact_id
JOIN 
    campaign ON campaign.id = campaign_contact.campaign_id
WHERE
    message.is_from_contact = TRUE AND
    campaign.organization_id = 5
```
### Text Leaderboard w/Formatted Dates
```SQL
select 
	u.id as user_id,
	u.last_name,
	u.first_name,
	u.email,
	u.cell,
	to_char(u.created_at AT time zone 'mst', 'FMDy, FMMon FMDD, YYYY at FMHH:MI PM') as join_date,
	u.alias,
	count (message.id) as Sent_Texts_all_time
from
	public.user AS u,
	public.message
JOIN
	campaign_contact ON campaign_contact.id = message.campaign_contact_id
JOIN campaign ON campaign.id = campaign_contact.campaign_id
WHERE
	message.user_id = u.id AND
  	message.is_from_contact = FALSE AND
	message.send_status = 'DELIVERED' AND
	campaign.organization_id = 5
group by
	u.id
order by
	u.id asc
```

### count answers of a specific answer
```SQL
SELECT
	interaction_step.id as answer_id,
	interaction_step.answer_option as answer,
	COUNT (question_response.id) AS answer_count
FROM
	public.interaction_step,
	public.question_response
WHERE
	question_response.interaction_step_id = interaction_step.id AND
	interaction_step.campaign_id = 37
GROUP BY
	interaction_step.id
ORDER BY
	interaction_step.id asc
```

### Kurt's Opt-Out Query
```SQL
SELECT external_id
FROM campaign_contact
INNER JOIN campaign
ON campaign.id = campaign_contact.campaign_id
LEFT OUTER JOIN opt_out
ON campaign_contact.cell = opt_out.cell
WHERE 
	(campaign.title LIKE '%PERSUASION%' or campaign.title LIKE '%Persuasion%')
	AND campaign.is_started = 'TRUE'
	AND is_opted_out = 'FALSE' 
	AND message_status = 'messaged'
	AND error_code IS NULL
	AND opt_out.id IS NULL
	AND external_id NOT IN (
		SELECT external_id
		FROM campaign_contact
		INNER JOIN campaign
		ON campaign.id = campaign_contact.campaign_id
		WHERE 
			(campaign.title LIKE '%PERSUASION%' OR 
		 	 campaign.title LIKE '%Persuasion%')
			AND campaign.is_started = 'TRUE'
			AND (message_status = 'closed' OR 
				 message_status = 'convo'  OR
				 message_status = 'needsResponse') 
	)
GROUP BY external_id
```
