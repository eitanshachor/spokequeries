# spokequeries
An updating list for keeping Spoke queries.

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

