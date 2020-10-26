# spokequeries
An updating list for keeping Spoke queries.

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
