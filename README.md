# Frontend Тестовое — QuestionCard

---

## Часть 1. Архитектура компонента

```
QuestionCard
├── QuestionStem
│   └── TipTapRenderer (с KaTeX)
├── AnswerOptions
│   └── AnswerOption[] (radio buttons)
├── ActionBar
│   ├── CheckAnswerButton
│   ├── MarkForReviewButton
│   └── NavigationButtons (prev/next)
└── ExplanationPanel (conditional)
    └── TipTapRenderer
```

### Где хранится состояние:

**Локально (внутри QuestionCard):**
- `selectedAnswer: string | null` — ID выбранного ответа
- `isChecked: boolean` — нажата ли кнопка Check Answer
- `isMarkedForReview: boolean` — отмечен ли вопрос для повторения
- `renderKey: number` — для принудительного ре-рендера при смене вопроса

**Глобально (Zustand store):**
- `currentQuestionId: string` — текущий вопрос
- `userAnswers: Map<questionId, {answer, isCorrect, timestamp}>` — история всех ответов
- `isDemoMode: boolean` — режим демо-доступа
- `markedQuestions: Set<questionId>` — вопросы для повторения

### Что сбрасывается при смене questionId:

```javascript
useEffect(() => {
  // Сбрасываем локальное состояние
  setSelectedAnswer(null)
  setIsChecked(false)
  setRenderKey(prev => prev + 1)
  
  // Загружаем сохраненный ответ если есть
  const savedAnswer = userAnswers.get(currentQuestionId)
  if (savedAnswer) {
    setSelectedAnswer(savedAnswer.answer)
    setIsChecked(true)
  }
}, [currentQuestionId])
```

**Не сбрасывается:**
- `userAnswers` (история)
- `markedQuestions` (глобальный список)
- `isDemoMode` (режим пользователя)

### Что будет при очень быстрых кликах:

**Проблема:** Race condition — пользователь нажал Check → сразу Next → API вернул результат для старого вопроса → UI показывает неверные данные.

**Решение:**
1. **Debounce на Check button** (300ms) — игнорируем повторные клики
2. **Disable всех кнопок** пока `isSubmitting === true`
3. **AbortController** — отменяем предыдущий запрос при смене вопроса
4. **Проверка актуальности** перед обновлением UI:
   ```javascript
   if (response.questionId !== currentQuestionId) return
   ```

---

## Часть 2. Псевдокод логики

```javascript
// === СОСТОЯНИЕ ===
selectedAnswer = null
isChecked = false
isSubmitting = false
isMarkedForReview = false
abortController = null

// === ВЫБОР ОТВЕТА ===
onSelectAnswer(answerId):
  if (isChecked || isSubmitting):
    return // нельзя менять после проверки
  
  selectedAnswer = answerId

// === CHECK ANSWER ===
onCheckAnswer():
  if (!selectedAnswer || isSubmitting || isChecked):
    return
  
  isSubmitting = true
  disableAllButtons()
  
  abortController = new AbortController()
  
  try:
    response = await api.checkAnswer({
      questionId: currentQuestionId,
      answer: selectedAnswer
    }, { signal: abortController.signal })
    
    // Проверка актуальности ответа
    if (response.questionId !== currentQuestionId):
      return // уже другой вопрос, игнорируем
    
    isChecked = true
    
    // Сохраняем в глобальное хранилище
    saveUserAnswer({
      questionId: currentQuestionId,
      answer: selectedAnswer,
      isCorrect: response.isCorrect,
      timestamp: Date.now()
    })
    
  catch (error):
    if (error.name === 'AbortError'):
      return // запрос отменен
    
    showErrorToast("Не удалось проверить ответ. Попробуйте еще раз")
    isChecked = false
    
  finally:
    isSubmitting = false
    enableButtons()

// === MARK FOR REVIEW ===
onMarkForReview():
  isMarkedForReview = !isMarkedForReview
  toggleMarkedQuestion(currentQuestionId)

// === СМЕНА ВОПРОСА ===
onQuestionChange(newQuestionId):
  // Отменяем текущий запрос
  abortController?.abort()
  
  // Сбрасываем локальное состояние
  selectedAnswer = null
  isChecked = false
  isSubmitting = false
  renderKey += 1
  
  // Проверяем есть ли сохраненный ответ
  savedAnswer = getUserAnswer(newQuestionId)
  if (savedAnswer):
    selectedAnswer = savedAnswer.answer
    isChecked = true
  
  // Проверяем отмечен ли для повторения
  isMarkedForReview = isQuestionMarked(newQuestionId)

// === DISABLED СОСТОЯНИЯ ===
checkButtonDisabled = 
  !selectedAnswer || isChecked || isSubmitting

answerOptionsDisabled = 
  isChecked || isSubmitting

navigationDisabled = 
  isSubmitting

markForReviewDisabled = 
  isSubmitting

// === ПОКАЗ EXPLANATION ===
shouldShowExplanation = 
  isChecked && !isDemoMode

shouldShowDemoPaywall = 
  isChecked && isDemoMode
```

---

## Часть 3. Edge Cases и UX

### 1. **Explanation отсутствует**
**UI показывает:**
- Заголовок "Explanation" остается
- Текст: "Объяснение для этого вопроса пока недоступно"
- Кнопка "Report Issue" для связи с поддержкой
- Правильность ответа все равно показывается (зеленый/красный индикатор)

### 2. **В stem только формулы**
**UI показывает:**
- KaTeX рендерит формулы нормально
- Если TipTap возвращает пустой content, показываем skeleton loader
- Fallback: "Загрузка вопроса..." пока контент не придет
- Если контент так и не пришел → Error state с кнопкой "Reload"

### 3. **В stem очень длинный текст**
**UI показывает:**
- `max-height: 400px` для QuestionStem с вертикальным скроллом
- Плавный gradient fade внизу если текст обрезан
- Кнопка "Expand question" → открывает в модальном окне
- На mobile: полностью scrollable без ограничения высоты

### 4. **KaTeX упал с ошибкой**
**UI показывает:**
- ErrorBoundary ловит ошибку внутри KaTeX компонента
- Показываем сырой LaTeX текст в `<code>` блоке: `$\frac{x^2}{y}$`
- Иконка ⚠️ с тултипом "Formula rendering failed"
- Логируем в Sentry с полным содержимым формулы для дебага
- Остальной контент вопроса работает нормально

### 5. **Пользователь меняет ответ после check**
**UI показывает:**
- **Запрещено**: все radio buttons задизейблены (`disabled={isChecked}`)
- Выбранный ответ подсвечен:
    - Зеленым если правильный
    - Красным если неправильный
    - Правильный ответ показан зеленым в любом случае
- Tooltip при hover на disabled option: "Cannot change answer after checking"
- Кнопка "Try similar question" для практики

### 6. **Пользователь в demo режиме**
**UI показывает:**

**После нажатия Check Answer:**
- Explanation блок рендерится с `filter: blur(10px)` и `pointer-events: none`
- Overlay поверх с градиентом и контентом:
  ```
  🔒 Full explanations available with subscription
  
  Get unlimited access to:
  • Detailed explanations for all questions
  • Mock exams with real timing
  • Progress tracking and analytics
  
  [Upgrade to Premium] — $9.99/month
  ```
- Preview первых 2 строк explanation без blur (teaser)
- CTA кнопка ведет на `/pricing` с utm_source=question_explanation
- В футере QuestionCard: "2 of 5 free questions remaining today"

**Ограничения demo:**
- Нельзя mark for review (кнопка задизейблена с тултипом)
- Mock exam недоступен (redirect на paywall)
- После 5 вопросов в день → hard paywall

---

## Бонус: Дополнительные соображения

### Performance
- TipTap content мемоизируется через `useMemo`
- KaTeX рендерится только при изменении формулы
- Debounce на все API calls
- Optimistic UI для mark for review (мгновенная реакция)

### Accessibility
- Proper ARIA labels на radio buttons
- Keyboard navigation (arrows для выбора ответа, Enter для Check)
- Focus management при смене вопроса
- Screen reader announcements для результата проверки

### Mobile готовность
- Touch-friendly tap targets (min 44px)
- Логика полностью переиспользуется в React Native
- UI адаптируется через Tailwind breakpoints
- Вся бизнес-логика в кастомных хуках → легко портировать

---
