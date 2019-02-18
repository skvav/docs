## Пособие по разработке плагинов

### Предлагаемый вид конвертера операций.
Сначала общая часть, а потом обработки частных случаев в виде [parseCase1, ..., parseCaseN].some(parser => ...) . Это позволяет прервать обработку путем возвращение false из parser, если операция полностью обработана.

```JavaScript
export function convertTransaction (apiTransaction, account, accountsById) {  
  const invoice = apiTransaction.operationAmount && apiTransaction.operationAmount.amount && {  
    sum: parseDecimal(apiTransaction.operationAmount.amount),  
    instrument: parseInstrument(apiTransaction.operationAmount.currency.code)  
  }  
  if (!invoice || !invoice.sum) {  
    return null  
  }  
  if (parseMetalInstrument(apiTransaction.operationAmount.currency.code)) {  
    invoice.sum = invoice.sum / GRAMS_IN_OZ  
  }  
  const feeStr = _.get(apiTransaction, 'details.paymentDetails.commission.amount')  
  const fee = feeStr ? -parseDecimal(feeStr) : 0  
  const transaction = {  
    movements: [  
      {  
        id: apiTransaction.id,  
        account: { id: account.id },  
        invoice,  
        sum: null,  
        fee  
  }  
    ],  
    date: parseDate(apiTransaction.date),  
    hold: apiTransaction.state === 'AUTHORIZATION',  
    merchant: null,  
    comment: null  
  };  
  [  
    parseInnerTransfer,  
    parseSum,  
    parseCashReplenishment,  
    parseCashWithdrawal,  
    parseOuterIncomeTransfer,  
    parseOutcomeTransfer,  
    parsePayee  
  ].some(parser => parser(transaction, apiTransaction, account, accountsById))  
  return transaction  
}
```

### Типы операций
При конвертации операций обязательно нужно распознавать следующие типы:
- расход
- доход
- пополнение наличными (перевод с наличных на карту/счёт)
- снятие наличных (перевод с карты/счёта на наличные)
- внутренний перевод в рамках одного банковского аккаунта пользователя. К примеру перевод с карты1 на карту2
- входящий внешний перевод с карты/счёта другого банка либо другого аккаунта банка плагина
- исходящий внешний перевод на карту/счёт другого банка либо другого аккаунта банка плагина

### Общие правила обработки операций:
- обрабатывать исходную валюту и сумму операции, если валюта отличается от валюты счёта. К примеру вы знаете, что была операция расхода 5 евро (400 рублей) с рублевого счёта. Тогда соответствующее движение должно содержать invoice:
```JavaScript
{
  ...
  movements: [
    {
      id: 'transaction_id',
      account: { id: 'account_id' },
      invoice: {
        sum: -5,
        instrument: 'EUR'
      },
      sum: -400,
      fee: 0
    },
    ...
  ],
  ...
}
```

- Плательщик / получатель операции всегда добавляется в merchant, кроме случая внутренних переводов и кроме случаев, когда плательщика невозможно определить (в этих случаях merchant = null). По возможности, если плагин уверен, что делает правильно, нужно парсить строчку названия мерчанта и вытаскивать город и страну. Пример:
```JavaScript
{
  ...
  merchant: {
    title: 'МакДональдс',
    city: 'MOSCOW',
    country: 'RUS',
    location: null,
    mcc: 1234
  },
  ...
```
Если не знаем, как распарсить строку мерчанта, и при этом в ней присутствует город или страна, то передаем:
```JavaScript
{
  ...
  merchant: {
    fullTitle: '12345 McDonalds GATCHINA RUS',
    location: null,
    mcc: 1234
  },
  ...
```

- Комментарий. В комментарий пишем только важную информацию. По возможности избегаем общих слов. К примеру, "Расход с карты", "Доход по счету" - это плохие комментарии. "Начисление процентов за период 01.01 - 01.02.2019", "Комиссия за мобильный банк", "Саше за баню" - хорошие.
- При внешних переводах стараемся определить сторонний банк. Для этого используем функцию common/accounts/parseOuterAccountData()

#### Плательщик / получатель и комментарий для переводов
 
Придерживаемся следующих правил:
- Если это внешний перевод и плательщик / получатель известен, то ставить его в merchant.title и передавать, если возможно, обе части перевода. Даже, если плательщик - тот же самый человек, что и пользователь плагина, все равно его указываем. Потому как, если не найдется вторая часть перевода среди операций пользователя, то мы создадим расход/доход с данным плательщиком. Этого будет достаточно пользователю, чтобы понять, что это за операция.
```JavaScript
{
  ...
  movements: [
    {
      id: 'transaction_id',
      account: { id: 'account_id' },
      invoice: {
        sum: -5,
        instrument: 'EUR'
      },
      sum: -400,
      fee: 0
    },
    {
      id: null,
      account: { 
        type: 'ccard',
        instrument: 'EUR',
        company: null,
        syncIds: ['4723'] 
      },
      invoice: null,
      sum: 5,
      fee: 0
    }
  ],
  merchant: {
    title: 'Николай Николаевич',
    city: null,
    country: null,
    location: null,
    mcc: null
  },
  comment: null
  ...
```
- Если это внешний перевод и плательщик неизвестен, а есть только название банка и/или syncId, то передаем перевод и пишем доступную информацию в комментарий
```JavaScript
{
  ...
  movements: [
    {
      id: 'transaction_id',
      account: { id: 'account_id' },
      invoice: {
        sum: -5,
        instrument: 'EUR'
      },
      sum: -400,
      fee: 0
    },
    {
      id: null,
      account: { 
        type: 'ccard',
        instrument: 'EUR',
        company: { id: '3' },
        syncIds: ['4723'] 
      },
      invoice: null,
      sum: 5,
      fee: 0
    }
  ],
  merchant: null,
  comment: 'Перевод в сторонний банк'
  ...
```
- Если внутренний перевод, то писать в комментарий только важную информацию, если ее предоставляет банк. К примеру о комиссии либо курсе. Но не общие слова наподобие: "Перевод на другую карту" и т.п.
```JavaScript
{
  ...
  movements: [
    {
      id: 'transaction_id',
      account: { id: 'account_id' },
      invoice: {
        sum: -5,
        instrument: 'EUR'
      },
      sum: -400,
      fee: 0
    },
    {
      id: 'transaction_id_2',
      account: { id: 'account_id_2' },
      invoice: null,
      sum: 5,
      fee: 0
    }
  ],
  merchant: null,
  comment: 'Перевод на карту в евро по курсу 80 руб.'
  ...
}
```
