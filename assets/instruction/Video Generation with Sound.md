# Video Generation with Sound

![Scheme 1](../images/Scheme-1.png)

Ми робимо автоматизацію, в якій:
1) генеруємо **фото** за промтом,  
2) з фото генеруємо **відео**,  
3) генеруємо **звук** до відео за промтом.

Для цього використовуємо:
- **OpenAI API**: https://platform.openai.com/  
- **fal.ai**: https://fal.ai/

Перейди за посиланнями, створи **API key** і поповни баланс **мінімум на $5** на кожній платформі, щоб запити почали працювати.

Далі:
- якщо ти вперше на https://n8n.io/ — бери **trial на 14 днів** і працюй у Cloud;
- якщо trial вже використаний — або **переїжджаєш на свій сервер (self-hosted)**, або купуєш підписку n8n: приблизно **€24/міс + податок** (виходить близько **€28.80/міс**).

Після цього заходимо в n8n і починаємо створювати **ноди** (вузли, які виконують конкретні дії).




## Ноди (початок)
1. **Manual Trigger** — запуск автоматизації по кліку  
2. **AI / OpenAI / Message a model** — обробка промта та генерація текстових даних  
   - **Model:** обираємо потрібну модель (після додавання Credentials з’явиться вибір).  
   - **Credentials:** додаємо/обираємо **OpenAI API Key** (ключ зберігаємо в Credentials, не в самій ноді).  
   - **Role:** **User**.  
   - **Prompt:** використовуємо промт з файлу: [`Приклад промта (або використайте свій)`](../prompts/Video%20Generation%20with%20Sound.md).  
  **Важливо:** промт має повертати дані у потрібній структурі, щоб воркфлоу працював коректно — обов’язково повинні бути поля: `image_prompt`, `video_prompt`, `negative_prompt`, `music_promt`.
   - **Response format:** **JSON** → вставляємо схему нижче.
```json
{
	"type": "object",
	"additionalProperties": false,
	"properties": {
		"frames": {
			"type": "array",
			"minItems": 1,
			"items": {
				"type": "object",
				"additionalProperties": false,
				"properties": {
					"image_prompt": { "type": "string", "minLength": 1 },
					"video_prompt": { "type": "string", "minLength": 1 },
					"negative_prompt": { "type": "string", "minLength": 1 },
					"music_promt": { "type": "string", "minLength": 1 }
				},
				"required": [
					"image_prompt",
					"video_prompt",
					"negative_prompt",
					"music_promt"
				]
			}
		}
	},
	"required": ["frames"]
}
```

3. **HTTP Request** — створення фото за промтом (fal.ai)
```text
POST https://fal.run/wan/v2.6/text-to-image
Authorization
Key API_KEY_FAL_AI
```
```json
{
	"prompt": "{{ $json.output[0].content[0].text.frames[0].image_prompt }}",
	"negative_prompt": "{{ $json.output[0].content[0].text.frames[0].negative_prompt }}",
	"image_size": "portrait_16_9",
	"max_images": 1,
	"enable_safety_checker": true
}
```

4. **Merge** — об’єднуємо дані в один масив  
Input 1: **HTTP Request** (генерація фото) 
Input 2: **Message a model**

```text
Mode: Combine
Combine By: Position
Number of Inputs: 2
```

5. **HTTP Request** — створення відео по фото за промтом (fal.ai)
```text
POST https://queue.fal.run/fal-ai/kandinsky5-pro/image-to-video
Authorization
Key API_KEY_FAL_AI
```
```json
{
	"prompt": "{{ $json.output[0].content[0].text.frames[0].video_prompt }}",
	"image_url": "{{ $json.images[0].url }}",
	"resolution": "512P",
	"duration": "5s",
	"num_inference_steps": 28,
	"acceleration": "regular"
}
```

![Scheme-wait-1](../images/Scheme-wait-1.png)

6. **Повторюваний блок (4 ноди): Wait → HTTP Request (status) → IF → HTTP Request (result)**  
Цей блок буде використовуватись **3 рази** в проєкті (для різних дій: відео/звук/склейка).

```text
Логіка:
1) Wait — чекаємо потрібний час
2) HTTP Request (GET) — перевіряємо статус задачі по status_url
3) IF — якщо статус COMPLETED → йдемо далі, якщо ні → повертаємось на Wait
4) HTTP Request (GET) — забираємо результат по response_url і передаємо в наступну ноду

1. Рекомендовані затримки (приклад):
- Генерація відео: 60s
- Генерація звука: 10s
- Склейка відео + звук: 15s

2. HTTP Request — status
Method: GET
URL: {{ $json.status_url }}

3. IF — перевірка статусу
{{ $json.status }} == "COMPLETED"

TRUE → до "HTTP Request — result"
FALSE → назад до "Wait"

4. HTTP Request — result
Method: GET
URL: {{ $json.response_url }}
```

7. **HTTP Request** — створення аудіо за промтом (fal.ai)  
Підключаємо після **Message a model**.

```text
POST https://queue.fal.run/cassetteai/sound-effects-generator
Authorization
Key API_KEY_FAL_AI
```
Якщо такий варіант кине помилку змінити його на Using Field Below і явно вказати значення
```json
{
	"prompt": "{{ $json.output[0].content[0].text.frames[0].music_promt }}",
	"duration": 5
}
```

8. Повторити крок **6**

9. **Merge** — об’єднуємо дані в один масив із крока **8** та **6**
Input 1: **8**
Input 2: **6**

```text
Mode: Combine
Combine By: Position
Number of Inputs: 2
```

10. **HTTP Request** — об’єднуємо авдіо та відео в один файл(fal.ai)
```text
POST https://queue.fal.run/fal-ai/ffmpeg-api/merge-audio-video
Authorization
Key API_KEY_FAL_AI
```
Якщо такий варіант кине помилку змінити його на Using Field Below і явно вказати значення
```json
{
	"video_url": "{{ $json.video.url }}",
	"audio_url": "{{ $json.audio_file.url }}",
	"start_offset": 0
}
```

11. Повторюємо крок **6** запускаєм нашу автоматизацію і при завершенні вибираєм активний алемент він нам і покаже наше відео 