1. 假設今天某個系統的前端UI要大改版，並且想要做A/B TEST，如果分成兩個git branch來開發舊版跟新版的UI，那後續如果系統有新功能or共同的bug要修改，要如何同時把這些改動同步到兩個branch？還是說這邊的開發人力成本就是直接變成兩倍？但因為新功能or共同的bug，其實邏輯的部分應該是可以抽出來，只是套到不同的UI上面而已，如果兩邊的開發人力沒有同步，就很有可能變成重工？

2. VMS當初的規劃的時候，由於圖表跟問卷這邊的資料結構變動的很頻繁，並且使用者的需求也可能會一直變動，當初在規劃DB Schema的時候，有兩個欄位（data, properties）是設計成JSON格式，前端丟什麼JSON，後端就直接把這個JSON存進DB。但如果data, properties的JSON格式有變動，那些原本在DB的舊資料，因為後端不會知道data, properties的JSON格式長怎樣，如果要撈出來一筆一筆更新，還要停機，成本太高，所以我們是採取被動更新，當那筆資料有被使用者Query，在前端的JavaScript就會有一個function validator，可以把舊的資料轉成新的資料，等到後續使用者如果有更新資料，就會把新的資料丟給後端，這樣就可以確保使用者不會因為拿到舊的資料，導致前端出錯，也可以確保熱資料有正確的被更新到DB。

假設原本的 `Survey` 長這樣
```ts
type Survey = {
  properties: {
    surveyPage: {
      banner: {
        img: string;
        width: number;
        height: number;
      };
    };
  };
};
```
後來使用者許願，希望banner圖片可以分成MOB, PAD, PC三種尺寸，分別上傳三張圖片，於是我就把properties的資料結構改成如下：
```ts
type Survey = {
  properties: {
    surveyPage: {
      banner: {
        mob: {
          img: string;
          width: number;
          height: number;
        };
        pad: {
          img: string;
          width: number;
          height: number;
        };
        pc: {
          img: string;
          width: number;
          height: number;
        };
      };
      /**
       * @deprecated since vX.X.X YYYY/MM/DD
       */
      img?: string;
      /**
       * @deprecated since vX.X.X YYYY/MM/DD
       */
      width?: number;
      /**
       * @deprecated since vX.X.X YYYY/MM/DD
       */
      height?: number;
    };
  };
};
```
這邊的資料結構改動其實是一個break change，舊資料不相容於新版的前端邏輯，為了避免前端跳出Client Side Error，所以我在前端加了一個 `function validator`，具體實作如下：
```ts
function validateSurvey (survey: Survey): Survey {
  if (!survey.properties.banner.mob) {
    survey.properties.banner.mob = {
      img: survey.properties.banner.img,
      width: survey.properties.banner.width,
      height: survey.properties.banner.height,
    };
  };
  if (!survey.properties.banner.pad) {
    survey.properties.banner.pad = {
      img: survey.properties.banner.img,
      width: survey.properties.banner.width,
      height: survey.properties.banner.height,
    };
  };
  if (!survey.properties.banner.pc) {
    survey.properties.banner.pc = {
      img: survey.properties.banner.img,
      width: survey.properties.banner.width,
      height: survey.properties.banner.height,
    };
  };
  delete survey.properties.banner.img;
  delete survey.properties.banner.width;
  delete survey.properties.banner.height;
  return survey;
}
```
問題如下：隨著時間拉長，使用者的需求會一直更改，這隻 `function validator` 就會越長越大，因為不確定DB資料是否都有更新到，所以這個 `function validator` 也只能一直掛在 production 環境。想問一下，有沒有更好的解法？或是目前的解法其實也可以應付了？

3. VMS問卷，當時新增了一個 `POST /login` 的API，本地端、測試環境串接都正常。結果程式部署到正式環境後，因為Authentication方式不一樣，本地端、測試環境都是用 `Authorization: Bearer {{token}}`，但正式環境，由於是掛在公司的domain，使用者ID是存在cookie，所以後端會去讀cookie，這段讀取cookie的程式有錯，導致後端一直回http status 401給前端，導致使用者一直卡在登入流程。目前有想到一個解法是，API部署到正式環境後，可以先用POSTMAN測過、前端這邊可以先用F12 Console直接發個fetch request的方式測，兩邊測過都沒問題之後，再把前端對應的版本部署上。想問除了這招，有沒有其他Solution？

4. 以後端來說，如果API的回傳格式有做出不相容的改動，例如原本回傳的是 `{ key: value }`，後來改成 `{ modifiedKey: value }`，這時候程式的主版號就要+1，例如 `V1.0.0 => V2.0.0`。那以前端來說，什麼時候會是一個主版號+1的時機？另外，這個版本號是面向使用者，還是面向開發者？以我開發VMS的經驗為例，我的主版號只有經歷過一次+1，那次是前端跟後端串接API的格式大調整，但其實如果面向使用者來看的話，這樣的改動對使用者是無感的，因為使用者並不會知道程式背後做了什麼事情，使用者只會看到畫面上的改動、新增了那些功能，所以前端的版本號到底該怎麼訂呢？還是說面向使用者是一套版本號，給內部的團隊開發者又是另一套版本號？但這樣維護上的成本就會變成雙倍？

5. VMS問卷這邊有個 "送出問卷填答" 的API `POST /answers`，使用者後來提了一個需求，希望這些問卷填答可以即時同步到 Google Spread Sheet，這樣行銷單位的同仁就可以即時看到問卷填答的狀態。這邊可以撰寫 Google Apps Script，把 "問卷填答寫進 Google Spread Sheet" 這段的 function 寫在 Google Apps Script，最終其實就是一個 `POST https://script.google.com/macros/s/{{id}}/exec` 的API Endpoint可以直接呼叫，呼叫之後就可以執行 "問卷填答寫進Google Spread Sheet" 的這段 function。
問題來了，最簡單的方式就是前端在呼叫 `POST /answers` 的時候，後端會先把問卷填答寫進DB，成功後再執行 `POST https://script.google.com/macros/s/{{id}}/exec`，這樣問卷填答就可以即時同步到 Google Spread Sheet，但缺點就是，對前端來說，API Response Time 會變長，因為 `POST https://script.google.com/macros/s/{{id}}/exec` 這段的時間是不可控的。如果以NodeJS的特性來開發的話，我可以很輕鬆地控制這個流程，讓前端先收到Response，不要等待 `POST https://script.google.com/macros/s/{{id}}/exec` 執行完，範例程式如下：

```ts
app.post('/answers', async (req, res) => {
  const body = req.body;
  // do some body validation here...
  const validatedBody = validateBody(body);

  // db insert
  const insertResult = await db.insertOne(validatedBody);

  // handle db insert failed...
  if (!insertResult.success) return res.status(500).send();

  // db insert success, before calling google api, send http status 200 to client first.
  res.status(200).send('ok');
  const responseFromGoogle = await fetch('https://script.google.com/macros/s/{{id}}/exec', {
    method: 'POST',
    body: validatedBody,
    headers: { Authorization: `Bearer {{token}}` }
  });

  if (responseFromGoogle.status === 200) // handle responseFromGoogle
});
```

但這個案子的後端是PHP，以我對PHP的認識，要做到像NODEJS這樣，先回200給前端，後續再處理 `POST https://script.google.com/macros/s/{{id}}/exec` 的話，應該是要額外裝一些非同步的套件，維護成本上又會更複雜。想問一下，如果今天的目標是要達成使用者填完問卷後，問卷填答可以即時同步到 Google Spread Sheet，以上述NODEJS的程式，是一個好的Solution嗎？如果是像PHP這種程式語言，假設我在 `POST https://script.google.com/macros/s/{{id}}/exec` 設了N秒的TIMEOUT，萬一只是因為Google Apps Script的反應時間比較慢，那該筆問卷填答就沒有同步到Google Spread Sheet，後續要怎麼回補資料？

6. 某個活動案子，讓使用者在1/1 ~ 1/10，每天簽到一次，每天的0000開放簽到，簽滿5次就可以獲得獎勵，只需要有登入即可簽到。當時規劃的簽到API很單純，就是一個 `POST /signIn`。但問題來了，這東西在本地端要怎麼測試？前端工程師在本地端測試，總不可能每天簽到一格吧？應該是需要有一個方法把簽到記錄刪掉，或是暫時跳過時間的限制，具體來說會怎麼做？我們當初的做法比較繞圈圈，我們在DB開了三個欄位
```
fakeCurrentDate
fakeStartDate
fakeEndDate
```
並且跟後端工程師討論好，開了對應的三隻API可以修改這三個欄位的值，並且再開一隻可以刪除個人所有簽到記錄的API，總共如下：
```
PUT /fakeCurrentDate
PUT /fakeStartDate
PUT /fakeEndDate
DELETE /signIn
```
活動上線前，就可以透過這四隻API達到重複簽到的效果
待活動正式上線，就把這四隻API拔掉，並且DB這三個欄位拔掉
程式端就把 `startDate` 跟 `endDate` 寫死，`currentDate` 就改成讀取現在的時間
想問一下，這種有時間區間的活動，通常來說上線前會怎麼測試？以面向前端工程師的角度，API要怎樣設計才方便前端工程師重複測試？以面向PM、公司內部員工的角度，要怎麼樣設計才方便內部使用者測試？

7. 為了增加我們公司訂閱制產品的銷量、同時增加使用者的黏著度，行銷部門推出了一個行銷活動案，新訂閱者前7天，只要每天來使用我們家的訂閱制產品，就可以簽到一次（一樣是每天只能簽到一次，每天0000重置簽到），7天內簽滿3次，就可以獲得獎勵。這種類型的活動，要怎麼測試？因為有牽扯到EC訂單的部分，訂閱成功後，EC那塊(是其他部門負責)會把新訂戶的會員ID跟訂閱時間透過API丟給我們部門，這邊先定義一下，那隻API的格式是 `POST /subscriber { userID: string; subscribeDate: string }`。我們就可以把這些新訂戶蒐集下來，後續處理活動7天內簽滿3次的邏輯。只是訂閱成功後的API發送，這塊就真的只能請內部同仁實際去刷卡測試了嗎（？這個案子跟上面那題有點類似，但我們這次沒有用 `fakeCurrentDate`, `fakeStartDate`, `fakeEndDate` 這種繞圈圈的做法，我們就是直接在上線前，讓PM走正規的程序去訂閱我們公司的訂閱制產品，走完完整的簽到流程，其餘內部同仁，則是收集他們的userID，直接灌進DB裡面，讓他們不用去訂閱我們公司的訂閱制產品，也可以直接測試7天內簽滿3次，就可以獲得獎勵的流程。

之所以這次活動不走上次那條
```
PUT /fakeCurrentDate
PUT /akeStartDate
PUT /fakeEndDate
DELETE /signIn
```
很繞圈圈的方式是因為，如果從專案管理的角度來看的話，開發這些API如果需要5天，那不如不要開發，直接提早5天上線讓PM走正規流程測試，也是因為這個案子才意識到，原來寫程式不是單純埋首苦幹，也要從專案管理的角度來看全局。

8. 關聯式資料庫，DB Schema的更新，這部份會建議直接把 `xxx.sql` 包進專案一起推到remote repository，然後啟動的時候再用 `xxx.sql` 去更新DB Schema嗎？還是說要用手動的方式連進DB，修改DB Schema？如果用手動的方式，要怎麼知道後端的哪個版本號，對應的DB Schema是什麼？如果production的程式有BUG，需要緊急退版，那DB Schema也要跟著回溯？因為我們目前會需要修改DB Schema的場景，其實都不會很複雜，通常都是加個欄位，可以向下兼容的改動，所以後端工程師目前都是手動連進測試機跟正式機的DB修改Schema，改好之後再把新版本的後端程式部署上去，這樣就不會有問題了。只是之前有發生過，新版的後端程式部署到正式環境，但是DB Schema忘記修改，導致production環境出錯，所以後來我們在發布每個後端版本的時候，都會同步更新一份視覺化的DB關聯表，就可以知道目前的DB Schema是對應後端的哪個版本了。只是我在想，如果把 `xxx.sql` 也包進專案一起推到remote repository的話，其實就可以跟著後端的版本號，很清楚的知道哪個版本新增了哪些功能，對應DB加了哪些Table，新增了哪些欄位等等。

9. VMS的圖片，會把圖片的URL、寬度、高度、名稱存在DB，原始的圖片則放在AWS S3，前端拿到圖片的URL、寬度、高度之後，就可以正常的渲染出一張圖片。今天要實作一個刪除圖片的API `DELETE /images/{{imageName}}`，要如何確保DB的資料跟AWS S3的資料都有被刪掉？最差的情況是，S3的圖片刪了，但是DB的資料沒刪，導致前端看到死圖。為了避免這個情況，程式端的執行順序是先刪除DB資料，確認刪除後，再執行S3圖片刪除，這樣可以確保，即便S3刪除圖片失敗，也只是S3多了一張用不到的圖片，不影響應用端。

更：後來想到另一個方法，是不是可以用MySQL Transaction的方式，確保DB跟S3都有刪除到，這方法是不是比較好？
``` ts
try {
  mysql.begin();
  mysql.deleteOne();
  mysql.commit();
  s3.deleteOne();
} catch (e) {
  mysql.rollback();
}
```

10. 如果說以某個CMS的使用場景為例，登入完之後，會需要立即取得使用者的資訊，並且根據使用者的權限，顯示不同的畫面。假設這是一個前後端分離的架構，那登入的API `POST /login` 跟取得使用者資訊的API `GET /user/{{id}}`，分成兩個API Endpoint會比較好，還是可以在登入的API成功之後，就把前端需要的使用者資訊回傳呢？API在設計的時候，是要以前端的需求為主，還是以後端方便操作維護為主？

11. MySQL如果設定一個欄位是VARCHAR(36) + ALLOW NULL + DEFAULT NULL，跟我直接設定VARCHAR(36) + DEFAULT 空字串，差在哪裡？

12. 這裡有個簡單的react app，請問你會怎麼優化？
``` ts
function ParentComponent () {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(!open)}>打開dialog</button>
      <dialog open={open}>dialog content here...</dialog>
      <ExpensiveComponent></ExpensiveComponent>
    </>
  )
}

function ExpensiveComponent () {
  // some expensive calculate here...
  return <div></div>
}
```

13. 操作BigQuery的時候，踩了一個坑，剛新增到BigQuery的資料，如果要修改or刪除的話，會報以下錯誤：
```
UPDATE or DELETE statement over table {{datasetID}}.{{tableID}} would affect rows in the streaming buffer, which is not supported
```
讓我不禁好奇，BigQuery會這樣設計的原因是？

14. 操作BigQuery的時候，踩了一個坑，短時間內執行大量的DML，會報以下錯誤：
```
ApiError: Exceeded rate limits: too many table dml insert operations for this table. For more information, see https://cloud.google.com/bigquery/docs/troubleshoot-quotas
```
實際的應用場景為，我們需要一次更新1個月內有註冊、修改資料的會員名單，所以才會在短時間執行大量的DML，但卻被BigQuery的安全政策擋了下來，後來的解法是，限制使用者批次上傳會員名單的量體，並且程式端在串接BigQuery API的時候，會一筆一筆執行，中間間隔N秒。

15. 問卷填答的資料結構會怎麼設計？

16. 公司有個大會員系統，所有網站的註冊資料，都是要串接這個大會員系統，但這樣也會有個問題，使用者若在A網站註冊，其實可以用該帳密直接在B網站登入，並且在B網站修改會員資料。如果A網站想要取得從A網站註冊的會員名單最新資料，就必須串接大會員系統的API，偏偏大會員系統只有設定 `GET /user/{id}` 的API，一次只能取得一筆會員資料，如此超沒效率。這種公司大架構上的問題，如果不是一個強而有力的CTO跳出來解決問題，只是一個部門的工程師的話，不太可能會有這種權力去推動改革，這種問題的話要怎麼解會比較好？