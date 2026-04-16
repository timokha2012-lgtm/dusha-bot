const https = require('https');

const TOKEN = process.env.BOT_TOKEN;
const ADMIN_ID = process.env.ADMIN_ID; // твой Telegram ID
const ANTHROPIC_KEY = process.env.ANTHROPIC_KEY;

let offset = 0;
const sessions = {};

const QUESTIONS = [
  {
    id: 'q1',
    text: `Привет. Я — бот Дьякона-психолога и душепопечителя.

Здесь не будет тестов на 100 вопросов.
Только несколько честных.

Готов? Тогда начнём.

*Вопрос 1 из 5*
Когда тебе плохо — что ты делаешь чаще всего?`,
    options: [
      'Помогаю другим — так легче',
      'Держусь и делаю вид что всё нормально',
      'Молчу и терплю',
      'Пытаюсь всё взять под контроль',
      'Опускаю руки — всё бесполезно'
    ]
  },
  {
    id: 'q2',
    text: `*Вопрос 2 из 5*
Как часто ты чувствуешь, что живёшь не своей жизнью?`,
    options: [
      'Почти всегда',
      'Часто',
      'Иногда',
      'Редко'
    ]
  },
  {
    id: 'q3',
    text: `*Вопрос 3 из 5*
Что тебе труднее всего?`,
    options: [
      'Сказать "нет" без чувства вины',
      'Попросить помощь',
      'Признать что мне плохо',
      'Отпустить контроль',
      'Поверить что я достоин любви'
    ]
  },
  {
    id: 'q4',
    text: `*Вопрос 4 из 5*
Есть ли в твоей жизни что-то, что ты давно хочешь изменить — но не меняешь?`,
    options: [
      'Да, и я знаю что именно',
      'Да, но не понимаю с чего начать',
      'Я устал пытаться',
      'Мне кажется уже поздно'
    ]
  },
  {
    id: 'q5',
    text: `*Вопрос 5 из 5*
Что тебя привело сюда сегодня?`,
    options: [
      'Хочу разобраться в себе',
      'Устал жить как раньше',
      'Проблемы в отношениях',
      'Зависимость — своя или близкого',
      'Тревога, депрессия, пустота',
      'Духовный кризис'
    ]
  }
];

const ROLES = {
  0: { name: 'Спасатель', desc: 'Ты живёшь для других и теряешь себя' },
  1: { name: 'Сильный', desc: 'Ты держишься изо всех сил и не даёшь себе упасть' },
  2: { name: 'Молчащий', desc: 'Ты несёшь всё внутри и не знаешь как выйти' },
  3: { name: 'Контролёр', desc: 'Ты держишь всё в руках потому что боишься хаоса' },
  4: { name: 'Потерявшийся', desc: 'Ты устал и не знаешь где ты настоящий' }
};

function apiRequest(hostname, path, data, extraHeaders = {}) {
  return new Promise((resolve, reject) => {
    const body = JSON.stringify(data);
    const req = https.request({
      hostname, path, method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Content-Length': Buffer.byteLength(body),
        ...extraHeaders
      }
    }, res => {
      let d = '';
      res.on('data', c => d += c);
      res.on('end', () => {
        try { resolve(JSON.parse(d)); } catch(e) { resolve({}); }
      });
    });
    req.on('error', reject);
    req.write(body);
    req.end();
  });
}

function tg(method, data) {
  return apiRequest('api.telegram.org', `/bot${TOKEN}/${method}`, data);
}

function claude(prompt) {
  return apiRequest(
    'api.anthropic.com', '/v1/messages',
    {
      model: 'claude-sonnet-4-20250514',
      max_tokens: 800,
      messages: [{ role: 'user', content: prompt }]
    },
    { 'x-api-key': ANTHROPIC_KEY, 'anthropic-version': '2023-06-01' }
  );
}

async function sendMessage(chatId, text, keyboard = null) {
  const data = { chat_id: chatId, text, parse_mode: 'Markdown' };
  if (keyboard) data.reply_markup = { keyboard, resize_keyboard: true, one_time_keyboard: true };
  return tg('sendMessage', data);
}

async function sendQuestion(chatId, questionIndex) {
  const q = QUESTIONS[questionIndex];
  const keyboard = q.options.map(o => [{ text: o }]);
  await sendMessage(chatId, q.text, keyboard);
}

async function generateAnalysis(answers) {
  const answerText = answers.map((a, i) => `Вопрос ${i+1}: ${a}`).join('\n');
  const roleIndex = answers[0] ? ['Помогаю другим', 'Держусь', 'Молчу', 'Контроль', 'Опускаю руки'].findIndex(r => answers[0].includes(r.split(' ')[0])) : 0;
  const role = ROLES[roleIndex >= 0 ? roleIndex : 0];

  const result = await claude(`Ты — дьякон-психолог и душепопечитель. Человек ответил на вопросы:

${answerText}

Напиши персональный разбор в 4-5 абзацах:
1. Назови его паттерн мягко и точно (он живёт как ${role.name} — ${role.desc})
2. Что за этим стоит психологически
3. Библейская мысль которая говорит именно к этой боли
4. Что можно сделать — первый шаг
5. Мягкое приглашение на бесплатную 30-минутную сессию

Стиль: прямой, живой, без пафоса. Обращение на "ты". Без списков — только абзацы.`);

  return result.content ? result.content.map(i => i.text || '').join('') : 'Спасибо за честность. Напишу тебе лично.';
}

async function notifyAdmin(userId, username, name, answers) {
  if (!ADMIN_ID) return;
  const text = `🔔 *Новый клиент в воронке*\n\nИмя: ${name}\nUsername: @${username || 'нет'}\nID: ${userId}\n\nОтветы:\n${answers.map((a,i) => `${i+1}. ${a}`).join('\n')}`;
  await tg('sendMessage', { chat_id: ADMIN_ID, text, parse_mode: 'Markdown' });
}

async function processMessage(msg) {
  const chatId = msg.chat.id;
  const userId = msg.from.id;
  const text = msg.text || '';
  const name = msg.from.first_name || 'Друг';
  const username = msg.from.username || '';

  if (!sessions[userId]) sessions[userId] = { step: -1, answers: [] };
  const session = sessions[userId];

  if (text === '/start' || text === 'Начать заново') {
    sessions[userId] = { step: 0, answers: [] };
    await sendQuestion(chatId, 0);
    return;
  }

  if (text === '/help') {
    await sendMessage(chatId, 'Напиши /start чтобы начать.\n\nПо вопросам пиши напрямую: @dusha_popechitel');
    return;
  }

  if (session.step >= 0 && session.step < QUESTIONS.length) {
    session.answers.push(text);
    session.step++;

    if (session.step < QUESTIONS.length) {
      await sendQuestion(chatId, session.step);
    } else {
      await sendMessage(chatId, '⏳ Анализирую твои ответы...');
      
      try {
        const analysis = await generateAnalysis(session.answers);
        await sendMessage(chatId, analysis);
        
        await new Promise(r => setTimeout(r, 1000));
        
        await sendMessage(chatId, 
          `Если хочешь разобраться глубже — я провожу *бесплатную 30-минутную сессию*.\n\nБез обязательств. Просто разговор.\n\nНапиши мне лично: @dusha_popechitel\nИли нажми кнопку ниже 👇`,
          [[{ text: '📩 Записаться на сессию' }], [{ text: 'Начать заново' }]]
        );

        await notifyAdmin(userId, username, name, session.answers);
        session.step = -1;
      } catch(e) {
        console.error('Ошибка генерации:', e.message);
        await sendMessage(chatId, 'Что-то пошло не так. Напиши мне напрямую: @dusha_popechitel');
      }
    }
    return;
  }

  if (text === '📩 Записаться на сессию') {
    await sendMessage(chatId, `Напиши мне напрямую в Telegram: @dusha_popechitel\n\nСкажи что прошёл опрос — я отвечу и мы договоримся о времени.`);
    await notifyAdmin(userId, username, name, ['Хочет записаться на сессию']);
    return;
  }

  await sendMessage(chatId, 'Напиши /start чтобы начать опрос 👇');
}

async function poll() {
  try {
    const res = await tg('getUpdates', { offset, limit: 100, timeout: 30 });
    if (res.ok && res.result && res.result.length > 0) {
      for (const update of res.result) {
        offset = update.update_id + 1;
        if (update.message) {
          await processMessage(update.message).catch(e => console.error('Ошибка обработки:', e.message));
        }
      }
    }
  } catch(e) {
    console.error('Polling error:', e.message);
  }
  setTimeout(poll, 1000);
}

console.log('🤖 Бот душепопечителя запущен');
poll();
