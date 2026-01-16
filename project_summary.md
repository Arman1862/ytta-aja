# Project Summary

## List Of Contents

- [.gitignore](#gitignore) 
- [app_summarry.md](#app_summarrymd) 
- [code.gs](#codegs) 
- [eslint.config.js](#eslintconfigjs) 
- [index.html](#indexhtml) 
- [package-lock.json](#packagelockjson) 
- [package.json](#packagejson) 
- [README.md](#readmemd) 
- [src/App.css](#srcappcss) 
- [src/App.jsx](#srcappjsx) 
- [src/components/Dashboard.jsx](#srccomponentsdashboardjsx) 
- [src/components/Footer.jsx](#srccomponentsfooterjsx) 
- [src/components/Home.jsx](#srccomponentshomejsx) 
- [src/components/KirimPesanAnonim.jsx](#srccomponentskirimpesananonimjsx) 
- [src/components/LoginForm.jsx](#srccomponentsloginformjsx) 
- [src/components/RegisterForm.jsx](#srccomponentsregisterformjsx) 
- [src/components/TampilPesanAnonim.jsx](#srccomponentstampilpesananonimjsx) 
- [src/config/api.js](#srcconfigapijs) 
- [src/index.css](#srcindexcss) 
- [src/main.jsx](#srcmainjsx) 
- [tailwind.config.cjs](#tailwindconfigcjs) 
- [tailwind.config.js](#tailwindconfigjs) 
- [vercel.json](#verceljson) 
- [vite.config.js](#viteconfigjs) 

---

## .gitignore 
```
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

node_modules
dist
dist-ssr
*.local

# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
.DS_Store
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?
```

---

## app_summarry.md
``markdown
### ?? Anonymous Q&A Inbox Project Concept Summary
**Objective** : To create a personalized, multi-user anonymous Q&A inbox system.

**Technology**: The project uses React for the user interface, styled with Tailwind CSS (Mobile First focus). The backend API and database are managed using Google Apps Script interfacing with Google Sheets.

**Design Style**: The design prioritizes a modern, clean look with Glassmorphism elements and high responsiveness.

### System Structure and Endpoints
The system divides functionality between the public frontend (React) and the self-managed backend (Apps Script).

Component / Function	Responsibility	Endpoint Type
RegisterForm.jsx	User Sign-up/Account Creation.	POST (action=register)
LoginForm.jsx	User Authentication.	GET (userId, loginKey)
KirimPesanAnonim.jsx	Anonymous Message Submission.	POST (action=send)
Dashboard.jsx	Displaying filtered inbox messages.	Receives data from doGet.

Ekspor ke Spreadsheet
Project Development Workflow
This workflow focuses on the sequential steps for both your responsibility (Apps Script/DB) and the AI agent's likely assistance area (ReactJS).

Step 1: Backend API Setup (User's Responsibility)

Google Sheets: You must set up two sheets: Users (with headers: UserID, LoginKey, NamaTampilan) and PesanAnonim (with headers: Tanggal, Pesan, Pengirim, RecipientID).

Apps Script (code.gs): You must implement the following core logic:

Registration (doPost?action=register): Check for unique UserID, generate LoginKey (UUID), and save to the Users sheet.

Login (doGet): Validate credentials, then filter PesanAnonim data by RecipientID.

Sending (doPost?action=send): Validate the existence of the RecipientID before saving the message.

Step 2: Frontend Implementation: Authentication Components

The AI agent can assist you in finalizing the RegisterForm.jsx and LoginForm.jsx components (which you will wire up to the endpoints you created in Step 1). These must handle loading states and display success/error messages (e.g., using SweetAlert2).

Step 3: Frontend Implementation: Core Interaction

The AI agent can assist in modifying KirimPesanAnonim.jsx to correctly read the recipientId from the URL parameter (?to=...) and include it in the POST payload (action=send).

The Dashboard.jsx component must be developed to consume the filtered data returned by your doGet function.

Step 4: Final Refinement

Implement React Router for smooth navigation.

Apply final Tailwind CSS refinements across all components, ensuring they are fully responsive and meet the Glassmorphism design standards.
``


---

## code.gs
``javascript
var sheetPesan = 'Pesan';
var sheetUsers = 'Users';
var scriptProp = PropertiesService.getScriptProperties();

function createJSONResponse(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}


function getOrCreateSheet(doc, sheetName, headers) {
  var sheet = doc.getSheetByName(sheetName);
  if (!sheet) {
    sheet = doc.insertSheet(sheetName);
    if (headers && headers.length > 0) {
      sheet.appendRow(headers);
    }
  }
  return sheet;
}

/**
 * Mencari data di sheet tertentu berdasarkan nilai kolom.
 * @returns {Array} Baris data yang cocok atau null.
 */
function findRowByValue(sheet, columnIndex, value) {
  if (!sheet) return null;
  var data = sheet.getDataRange().getValues();
  for (var i = 1; i < data.length; i++) {
    if (String(data[i][columnIndex]).toLowerCase() === String(value).toLowerCase()) {
      return data[i];
    }
  }
  return null;
}

/**
 * Membuat kunci login acak 6 digit.
 */
function generateLoginKey() {
  return Math.random().toString(36).substring(2, 8).toUpperCase();
}



/**
 * Menerima request GET dari Web App (untuk Login dan Ambil Pesan)
 */
function doGet(e) {
  var action = e.parameter.action;

  if (action === 'login') {
    var userId = e.parameter.userId;
    var loginKey = e.parameter.loginKey;

    if (!userId || !loginKey) {
      return createJSONResponse({'result': 'error', 'message': 'UserID dan Login Key wajib diisi.'});
    }

    try {
      var doc = SpreadsheetApp.getActiveSpreadsheet();
      var usersSheet = doc.getSheetByName(sheetUsers);

      var userRow = findRowByValue(usersSheet, 0, userId); 

      if (!userRow) {
        return createJSONResponse({'result': 'error', 'message': 'UserID tidak ditemukan.'});
      }

      if (String(userRow[1]) !== String(loginKey)) { 
        return createJSONResponse({'result': 'error', 'message': 'Login Key salah!'});
      }

      var userProfile = {
        userId: String(userRow[0]),
        loginKey: String(userRow[1]),
        namaTampilan: String(userRow[2] || '') 
      };

      var userLoggedInID = userProfile.userId.toLowerCase();


      var pesanSheet = doc.getSheetByName(sheetPesan);
      var messages = [];

      if (pesanSheet) {
          var dataRange = pesanSheet.getDataRange();
          var values = dataRange.getValues();
          
          var header = values.length > 0 ? values[0] : [];
          var recipientIdIndex = header.indexOf('RecipientID'); 

          for (var i = 1; i < values.length; i++) {
              var row = values[i];
              
              if (row[recipientIdIndex] && String(row[recipientIdIndex]).toLowerCase() === userLoggedInID) { 
                  messages.push({
                      Tanggal: row[0],
                      Pesan: row[1],
                      Pengirim: row[2],
                      RecipientID: row[3] 
                  });
              }
          }
      }

      return createJSONResponse({ 
        'result': 'success',
        'profile': userProfile,
        'messages': messages.reverse() 
      });

    } catch (e) {
      Logger.log('Error details (doGet - login): ' + e.toString());
      return createJSONResponse({'result': 'error', 'message': 'Terjadi kesalahan sistem saat login: ' + e.toString()});
    }
  } 

  return createJSONResponse({'result': 'error', 'message': 'Endpoint tidak valid.'});
}


/**
 * Menerima request POST dari Web App (untuk Register dan Kirim Pesan)
 */
function doPost(e) {
  var data = e.parameter;
  var action = data.action;

  try {
    var doc = SpreadsheetApp.getActiveSpreadsheet();

    if (action === 'register') {
      var userId = data.userId;
      var namaTampilan = data.namaTampilan;
      var loginKey = generateLoginKey();

      if (!userId || !namaTampilan) {
        return createJSONResponse({'result': 'error', 'message': 'UserID dan Nama Tampilan wajib diisi.'});
      }
      
      var usersSheet = getOrCreateSheet(doc, sheetUsers, ['UserID', 'LoginKey', 'NamaTampilan']);
      
      if (findRowByValue(usersSheet, 0, userId)) {
        return createJSONResponse({'result': 'error', 'message': 'UserID ' + userId + ' sudah digunakan.'});
      }

      usersSheet.appendRow([userId.trim(), loginKey, namaTampilan.trim()]);

      return createJSONResponse({
        'result': 'success',
        'message': 'Registrasi berhasil!',
        'data': {
          'userId': userId.trim(),
          'loginKey': loginKey
        }
      });

    } else if (data.pesan && data.recipientId) {
      var pesan = data.pesan;
      var pengirim = data.pengirim || 'Anonim';
      var recipientId = data.recipientId;
      var timestamp = new Date();

      var pesanSheet = getOrCreateSheet(doc, sheetPesan, ['Tanggal', 'Pesan', 'Pengirim', 'RecipientID']);

      var usersSheet = doc.getSheetByName(sheetUsers);
      if (!usersSheet || !findRowByValue(usersSheet, 0, recipientId)) {
        return createJSONResponse({'result': 'error', 'message': 'Recipient ID tidak ditemukan.'});
      }

      pesanSheet.appendRow([timestamp, pesan, pengirim, recipientId.trim()]);

      return createJSONResponse({'result': 'success', 'message': 'Pesan berhasil dikirim.'});
    }

    return createJSONResponse({'result': 'error', 'message': 'Parameter tidak lengkap atau action tidak valid.'});

  } catch (e) {
    Logger.log('Error details (doPost): ' + e.toString());
    return createJSONResponse({'result': 'error', 'message': 'Terjadi kesalahan sistem saat memproses data: ' + e.toString()});
  }
}
``


---

## eslint.config.js
``javascript
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import { defineConfig, globalIgnores } from 'eslint/config'

export default defineConfig([
  globalIgnores(['dist']),
  {
    files: ['**/*.{js,jsx}'],
    extends: [
      js.configs.recommended,
      reactHooks.configs['recommended-latest'],
      reactRefresh.configs.vite,
    ],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
      parserOptions: {
        ecmaVersion: 'latest',
        ecmaFeatures: { jsx: true },
        sourceType: 'module',
      },
    },
    rules: {
      'no-unused-vars': ['error', { varsIgnorePattern: '^[A-Z_]' }],
    },
  },
])
``


---

## index.html
``html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>YTTA-AJA</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
``


---

## package.json
``json
{
  "name": "ngl-remake",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^19.1.1",
    "react-bootstrap-icons": "^1.11.6",
    "react-dom": "^19.1.1",
    "react-router-dom": "^7.9.3",
    "sweetalert2": "^11.23.0"
  },
  "devDependencies": {
    "@eslint/js": "^9.36.0",
    "@tailwindcss/vite": "^4.1.13",
    "@types/react": "^19.1.13",
    "@types/react-dom": "^19.1.9",
    "@vitejs/plugin-react": "^5.0.3",
    "eslint": "^9.36.0",
    "eslint-plugin-react-hooks": "^5.2.0",
    "eslint-plugin-react-refresh": "^0.4.20",
    "globals": "^16.4.0",
    "tailwindcss": "^4.0.0",
    "vite": "^7.1.7"
  }
}
``


---

## package-lock.json
``json
{
  "name": "ngl-remake",
  "version": "0.0.0",
  "lockfileVersion": 3,
  "requires": true,
  "packages": {
    "": {
      "name": "ngl-remake",
      "version": "0.0.0",
      "dependencies": {
        "react": "^19.1.1",
        "react-bootstrap-icons": "^1.11.6",
        "react-dom": "^19.1.1",
        "react-router-dom": "^7.9.3",
        "sweetalert2": "^11.23.0"
      },
      "devDependencies": {
        "@eslint/js": "^9.36.0",
        "@tailwindcss/vite": "^4.1.13",
        "@types/react": "^19.1.13",
        "@types/react-dom": "^19.1.9",
        "@vitejs/plugin-react": "^5.0.3",
        "eslint": "^9.36.0",
        "eslint-plugin-react-hooks": "^5.2.0",
        "eslint-plugin-react-refresh": "^0.4.20",
        "globals": "^16.4.0",
        "tailwindcss": "^4.0.0",
        "vite": "^7.1.7"
      }
    },
    "node_modules/@babel/code-frame": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/code-frame/-/code-frame-7.27.1.tgz",
      "integrity": "sha512-cjQ7ZlQ0Mv3b47hABuTevyTuYN4i+loJKGeV9flcCgIK37cCXRh+L1bd3iBHlynerhQ7BhCkn2BPbQUL+rGqFg==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/helper-validator-identifier": "^7.27.1",
        "js-tokens": "^4.0.0",
        "picocolors": "^1.1.1"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/compat-data": {
      "version": "7.28.4",
      "resolved": "https://registry.npmjs.org/@babel/compat-data/-/compat-data-7.28.4.tgz",
      "integrity": "sha512-YsmSKC29MJwf0gF8Rjjrg5LQCmyh+j/nD8/eP7f+BeoQTKYqs9RoWbjGOdy0+1Ekr68RJZMUOPVQaQisnIo4Rw==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/core": {
      "version": "7.28.4",
      "resolved": "https://registry.npmjs.org/@babel/core/-/core-7.28.4.tgz",
      "integrity": "sha512-2BCOP7TN8M+gVDj7/ht3hsaO/B/n5oDbiAyyvnRlNOs+u1o+JWNYTQrmpuNp1/Wq2gcFrI01JAW+paEKDMx/CA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/code-frame": "^7.27.1",
        "@babel/generator": "^7.28.3",
        "@babel/helper-compilation-targets": "^7.27.2",
        "@babel/helper-module-transforms": "^7.28.3",
        "@babel/helpers": "^7.28.4",
        "@babel/parser": "^7.28.4",
        "@babel/template": "^7.27.2",
        "@babel/traverse": "^7.28.4",
        "@babel/types": "^7.28.4",
        "@jridgewell/remapping": "^2.3.5",
        "convert-source-map": "^2.0.0",
        "debug": "^4.1.0",
        "gensync": "^1.0.0-beta.2",
        "json5": "^2.2.3",
        "semver": "^6.3.1"
      },
      "engines": {
        "node": ">=6.9.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/babel"
      }
    },
    "node_modules/@babel/generator": {
      "version": "7.28.3",
      "resolved": "https://registry.npmjs.org/@babel/generator/-/generator-7.28.3.tgz",
      "integrity": "sha512-3lSpxGgvnmZznmBkCRnVREPUFJv2wrv9iAoFDvADJc0ypmdOxdUtcLeBgBJ6zE0PMeTKnxeQzyk0xTBq4Ep7zw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/parser": "^7.28.3",
        "@babel/types": "^7.28.2",
        "@jridgewell/gen-mapping": "^0.3.12",
        "@jridgewell/trace-mapping": "^0.3.28",
        "jsesc": "^3.0.2"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-compilation-targets": {
      "version": "7.27.2",
      "resolved": "https://registry.npmjs.org/@babel/helper-compilation-targets/-/helper-compilation-targets-7.27.2.tgz",
      "integrity": "sha512-2+1thGUUWWjLTYTHZWK1n8Yga0ijBz1XAhUXcKy81rd5g6yh7hGqMp45v7cadSbEHc9G3OTv45SyneRN3ps4DQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/compat-data": "^7.27.2",
        "@babel/helper-validator-option": "^7.27.1",
        "browserslist": "^4.24.0",
        "lru-cache": "^5.1.1",
        "semver": "^6.3.1"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-globals": {
      "version": "7.28.0",
      "resolved": "https://registry.npmjs.org/@babel/helper-globals/-/helper-globals-7.28.0.tgz",
      "integrity": "sha512-+W6cISkXFa1jXsDEdYA8HeevQT/FULhxzR99pxphltZcVaugps53THCeiWA8SguxxpSp3gKPiuYfSWopkLQ4hw==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-module-imports": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/helper-module-imports/-/helper-module-imports-7.27.1.tgz",
      "integrity": "sha512-0gSFWUPNXNopqtIPQvlD5WgXYI5GY2kP2cCvoT8kczjbfcfuIljTbcWrulD1CIPIX2gt1wghbDy08yE1p+/r3w==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/traverse": "^7.27.1",
        "@babel/types": "^7.27.1"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-module-transforms": {
      "version": "7.28.3",
      "resolved": "https://registry.npmjs.org/@babel/helper-module-transforms/-/helper-module-transforms-7.28.3.tgz",
      "integrity": "sha512-gytXUbs8k2sXS9PnQptz5o0QnpLL51SwASIORY6XaBKF88nsOT0Zw9szLqlSGQDP/4TljBAD5y98p2U1fqkdsw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/helper-module-imports": "^7.27.1",
        "@babel/helper-validator-identifier": "^7.27.1",
        "@babel/traverse": "^7.28.3"
      },
      "engines": {
        "node": ">=6.9.0"
      },
      "peerDependencies": {
        "@babel/core": "^7.0.0"
      }
    },
    "node_modules/@babel/helper-plugin-utils": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/helper-plugin-utils/-/helper-plugin-utils-7.27.1.tgz",
      "integrity": "sha512-1gn1Up5YXka3YYAHGKpbideQ5Yjf1tDa9qYcgysz+cNCXukyLl6DjPXhD3VRwSb8c0J9tA4b2+rHEZtc6R0tlw==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-string-parser": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/helper-string-parser/-/helper-string-parser-7.27.1.tgz",
      "integrity": "sha512-qMlSxKbpRlAridDExk92nSobyDdpPijUq2DW6oDnUqd0iOGxmQjyqhMIihI9+zv4LPyZdRje2cavWPbCbWm3eA==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-validator-identifier": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/helper-validator-identifier/-/helper-validator-identifier-7.27.1.tgz",
      "integrity": "sha512-D2hP9eA+Sqx1kBZgzxZh0y1trbuU+JoDkiEwqhQ36nodYqJwyEIhPSdMNd7lOm/4io72luTPWH20Yda0xOuUow==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helper-validator-option": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/helper-validator-option/-/helper-validator-option-7.27.1.tgz",
      "integrity": "sha512-YvjJow9FxbhFFKDSuFnVCe2WxXk1zWc22fFePVNEaWJEu8IrZVlda6N0uHwzZrUM1il7NC9Mlp4MaJYbYd9JSg==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/helpers": {
      "version": "7.28.4",
      "resolved": "https://registry.npmjs.org/@babel/helpers/-/helpers-7.28.4.tgz",
      "integrity": "sha512-HFN59MmQXGHVyYadKLVumYsA9dBFun/ldYxipEjzA4196jpLZd8UjEEBLkbEkvfYreDqJhZxYAWFPtrfhNpj4w==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/template": "^7.27.2",
        "@babel/types": "^7.28.4"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/parser": {
      "version": "7.28.4",
      "resolved": "https://registry.npmjs.org/@babel/parser/-/parser-7.28.4.tgz",
      "integrity": "sha512-yZbBqeM6TkpP9du/I2pUZnJsRMGGvOuIrhjzC1AwHwW+6he4mni6Bp/m8ijn0iOuZuPI2BfkCoSRunpyjnrQKg==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/types": "^7.28.4"
      },
      "bin": {
        "parser": "bin/babel-parser.js"
      },
      "engines": {
        "node": ">=6.0.0"
      }
    },
    "node_modules/@babel/plugin-transform-react-jsx-self": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/plugin-transform-react-jsx-self/-/plugin-transform-react-jsx-self-7.27.1.tgz",
      "integrity": "sha512-6UzkCs+ejGdZ5mFFC/OCUrv028ab2fp1znZmCZjAOBKiBK2jXD1O+BPSfX8X2qjJ75fZBMSnQn3Rq2mrBJK2mw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/helper-plugin-utils": "^7.27.1"
      },
      "engines": {
        "node": ">=6.9.0"
      },
      "peerDependencies": {
        "@babel/core": "^7.0.0-0"
      }
    },
    "node_modules/@babel/plugin-transform-react-jsx-source": {
      "version": "7.27.1",
      "resolved": "https://registry.npmjs.org/@babel/plugin-transform-react-jsx-source/-/plugin-transform-react-jsx-source-7.27.1.tgz",
      "integrity": "sha512-zbwoTsBruTeKB9hSq73ha66iFeJHuaFkUbwvqElnygoNbj/jHRsSeokowZFN3CZ64IvEqcmmkVe89OPXc7ldAw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/helper-plugin-utils": "^7.27.1"
      },
      "engines": {
        "node": ">=6.9.0"
      },
      "peerDependencies": {
        "@babel/core": "^7.0.0-0"
      }
    },
    "node_modules/@babel/template": {
      "version": "7.27.2",
      "resolved": "https://registry.npmjs.org/@babel/template/-/template-7.27.2.tgz",
      "integrity": "sha512-LPDZ85aEJyYSd18/DkjNh4/y1ntkE5KwUHWTiqgRxruuZL2F1yuHligVHLvcHY2vMHXttKFpJn6LwfI7cw7ODw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/code-frame": "^7.27.1",
        "@babel/parser": "^7.27.2",
        "@babel/types": "^7.27.1"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/traverse": {
      "version": "7.28.4",
      "resolved": "https://registry.npmjs.org/@babel/traverse/-/traverse-7.28.4.tgz",
      "integrity": "sha512-YEzuboP2qvQavAcjgQNVgsvHIDv6ZpwXvcvjmyySP2DIMuByS/6ioU5G9pYrWHM6T2YDfc7xga9iNzYOs12CFQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/code-frame": "^7.27.1",
        "@babel/generator": "^7.28.3",
        "@babel/helper-globals": "^7.28.0",
        "@babel/parser": "^7.28.4",
        "@babel/template": "^7.27.2",
        "@babel/types": "^7.28.4",
        "debug": "^4.3.1"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@babel/types": {
      "version": "7.28.4",
      "resolved": "https://registry.npmjs.org/@babel/types/-/types-7.28.4.tgz",
      "integrity": "sha512-bkFqkLhh3pMBUQQkpVgWDWq/lqzc2678eUyDlTBhRqhCHFguYYGM0Efga7tYk4TogG/3x0EEl66/OQ+WGbWB/Q==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/helper-string-parser": "^7.27.1",
        "@babel/helper-validator-identifier": "^7.27.1"
      },
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/@esbuild/aix-ppc64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/aix-ppc64/-/aix-ppc64-0.25.10.tgz",
      "integrity": "sha512-0NFWnA+7l41irNuaSVlLfgNT12caWJVLzp5eAVhZ0z1qpxbockccEt3s+149rE64VUI3Ml2zt8Nv5JVc4QXTsw==",
      "cpu": [
        "ppc64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "aix"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/android-arm": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/android-arm/-/android-arm-0.25.10.tgz",
      "integrity": "sha512-dQAxF1dW1C3zpeCDc5KqIYuZ1tgAdRXNoZP7vkBIRtKZPYe2xVr/d3SkirklCHudW1B45tGiUlz2pUWDfbDD4w==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "android"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/android-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/android-arm64/-/android-arm64-0.25.10.tgz",
      "integrity": "sha512-LSQa7eDahypv/VO6WKohZGPSJDq5OVOo3UoFR1E4t4Gj1W7zEQMUhI+lo81H+DtB+kP+tDgBp+M4oNCwp6kffg==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "android"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/android-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/android-x64/-/android-x64-0.25.10.tgz",
      "integrity": "sha512-MiC9CWdPrfhibcXwr39p9ha1x0lZJ9KaVfvzA0Wxwz9ETX4v5CHfF09bx935nHlhi+MxhA63dKRRQLiVgSUtEg==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "android"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/darwin-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/darwin-arm64/-/darwin-arm64-0.25.10.tgz",
      "integrity": "sha512-JC74bdXcQEpW9KkV326WpZZjLguSZ3DfS8wrrvPMHgQOIEIG/sPXEN/V8IssoJhbefLRcRqw6RQH2NnpdprtMA==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/darwin-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/darwin-x64/-/darwin-x64-0.25.10.tgz",
      "integrity": "sha512-tguWg1olF6DGqzws97pKZ8G2L7Ig1vjDmGTwcTuYHbuU6TTjJe5FXbgs5C1BBzHbJ2bo1m3WkQDbWO2PvamRcg==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/freebsd-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-arm64/-/freebsd-arm64-0.25.10.tgz",
      "integrity": "sha512-3ZioSQSg1HT2N05YxeJWYR+Libe3bREVSdWhEEgExWaDtyFbbXWb49QgPvFH8u03vUPX10JhJPcz7s9t9+boWg==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "freebsd"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/freebsd-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/freebsd-x64/-/freebsd-x64-0.25.10.tgz",
      "integrity": "sha512-LLgJfHJk014Aa4anGDbh8bmI5Lk+QidDmGzuC2D+vP7mv/GeSN+H39zOf7pN5N8p059FcOfs2bVlrRr4SK9WxA==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "freebsd"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-arm": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm/-/linux-arm-0.25.10.tgz",
      "integrity": "sha512-oR31GtBTFYCqEBALI9r6WxoU/ZofZl962pouZRTEYECvNF/dtXKku8YXcJkhgK/beU+zedXfIzHijSRapJY3vg==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-arm64/-/linux-arm64-0.25.10.tgz",
      "integrity": "sha512-5luJWN6YKBsawd5f9i4+c+geYiVEw20FVW5x0v1kEMWNq8UctFjDiMATBxLvmmHA4bf7F6hTRaJgtghFr9iziQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-ia32": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-ia32/-/linux-ia32-0.25.10.tgz",
      "integrity": "sha512-NrSCx2Kim3EnnWgS4Txn0QGt0Xipoumb6z6sUtl5bOEZIVKhzfyp/Lyw4C1DIYvzeW/5mWYPBFJU3a/8Yr75DQ==",
      "cpu": [
        "ia32"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-loong64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-loong64/-/linux-loong64-0.25.10.tgz",
      "integrity": "sha512-xoSphrd4AZda8+rUDDfD9J6FUMjrkTz8itpTITM4/xgerAZZcFW7Dv+sun7333IfKxGG8gAq+3NbfEMJfiY+Eg==",
      "cpu": [
        "loong64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-mips64el": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-mips64el/-/linux-mips64el-0.25.10.tgz",
      "integrity": "sha512-ab6eiuCwoMmYDyTnyptoKkVS3k8fy/1Uvq7Dj5czXI6DF2GqD2ToInBI0SHOp5/X1BdZ26RKc5+qjQNGRBelRA==",
      "cpu": [
        "mips64el"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-ppc64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-ppc64/-/linux-ppc64-0.25.10.tgz",
      "integrity": "sha512-NLinzzOgZQsGpsTkEbdJTCanwA5/wozN9dSgEl12haXJBzMTpssebuXR42bthOF3z7zXFWH1AmvWunUCkBE4EA==",
      "cpu": [
        "ppc64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-riscv64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-riscv64/-/linux-riscv64-0.25.10.tgz",
      "integrity": "sha512-FE557XdZDrtX8NMIeA8LBJX3dC2M8VGXwfrQWU7LB5SLOajfJIxmSdyL/gU1m64Zs9CBKvm4UAuBp5aJ8OgnrA==",
      "cpu": [
        "riscv64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-s390x": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-s390x/-/linux-s390x-0.25.10.tgz",
      "integrity": "sha512-3BBSbgzuB9ajLoVZk0mGu+EHlBwkusRmeNYdqmznmMc9zGASFjSsxgkNsqmXugpPk00gJ0JNKh/97nxmjctdew==",
      "cpu": [
        "s390x"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/linux-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/linux-x64/-/linux-x64-0.25.10.tgz",
      "integrity": "sha512-QSX81KhFoZGwenVyPoberggdW1nrQZSvfVDAIUXr3WqLRZGZqWk/P4T8p2SP+de2Sr5HPcvjhcJzEiulKgnxtA==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/netbsd-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-arm64/-/netbsd-arm64-0.25.10.tgz",
      "integrity": "sha512-AKQM3gfYfSW8XRk8DdMCzaLUFB15dTrZfnX8WXQoOUpUBQ+NaAFCP1kPS/ykbbGYz7rxn0WS48/81l9hFl3u4A==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "netbsd"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/netbsd-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/netbsd-x64/-/netbsd-x64-0.25.10.tgz",
      "integrity": "sha512-7RTytDPGU6fek/hWuN9qQpeGPBZFfB4zZgcz2VK2Z5VpdUxEI8JKYsg3JfO0n/Z1E/6l05n0unDCNc4HnhQGig==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "netbsd"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/openbsd-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-arm64/-/openbsd-arm64-0.25.10.tgz",
      "integrity": "sha512-5Se0VM9Wtq797YFn+dLimf2Zx6McttsH2olUBsDml+lm0GOCRVebRWUvDtkY4BWYv/3NgzS8b/UM3jQNh5hYyw==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "openbsd"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/openbsd-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/openbsd-x64/-/openbsd-x64-0.25.10.tgz",
      "integrity": "sha512-XkA4frq1TLj4bEMB+2HnI0+4RnjbuGZfet2gs/LNs5Hc7D89ZQBHQ0gL2ND6Lzu1+QVkjp3x1gIcPKzRNP8bXw==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "openbsd"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/openharmony-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/openharmony-arm64/-/openharmony-arm64-0.25.10.tgz",
      "integrity": "sha512-AVTSBhTX8Y/Fz6OmIVBip9tJzZEUcY8WLh7I59+upa5/GPhh2/aM6bvOMQySspnCCHvFi79kMtdJS1w0DXAeag==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "openharmony"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/sunos-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/sunos-x64/-/sunos-x64-0.25.10.tgz",
      "integrity": "sha512-fswk3XT0Uf2pGJmOpDB7yknqhVkJQkAQOcW/ccVOtfx05LkbWOaRAtn5SaqXypeKQra1QaEa841PgrSL9ubSPQ==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "sunos"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/win32-arm64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/win32-arm64/-/win32-arm64-0.25.10.tgz",
      "integrity": "sha512-ah+9b59KDTSfpaCg6VdJoOQvKjI33nTaQr4UluQwW7aEwZQsbMCfTmfEO4VyewOxx4RaDT/xCy9ra2GPWmO7Kw==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@esbuild/win32-ia32": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/win32-ia32/-/win32-ia32-0.25.10.tgz",
      "integrity": "sha512-QHPDbKkrGO8/cz9LKVnJU22HOi4pxZnZhhA2HYHez5Pz4JeffhDjf85E57Oyco163GnzNCVkZK0b/n4Y0UHcSw==",
      "cpu": [
        "ia32"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/@eslint-community/eslint-utils": {
      "version": "4.9.0",
      "resolved": "https://registry.npmjs.org/@eslint-community/eslint-utils/-/eslint-utils-4.9.0.tgz",
      "integrity": "sha512-ayVFHdtZ+hsq1t2Dy24wCmGXGe4q9Gu3smhLYALJrr473ZH27MsnSL+LKUlimp4BWJqMDMLmPpx/Q9R3OAlL4g==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "eslint-visitor-keys": "^3.4.3"
      },
      "engines": {
        "node": "^12.22.0 || ^14.17.0 || >=16.0.0"
      },
      "funding": {
        "url": "https://opencollective.com/eslint"
      },
      "peerDependencies": {
        "eslint": "^6.0.0 || ^7.0.0 || >=8.0.0"
      }
    },
    "node_modules/@eslint-community/eslint-utils/node_modules/eslint-visitor-keys": {
      "version": "3.4.3",
      "resolved": "https://registry.npmjs.org/eslint-visitor-keys/-/eslint-visitor-keys-3.4.3.tgz",
      "integrity": "sha512-wpc+LXeiyiisxPlEkUzU6svyS1frIO3Mgxj1fdy7Pm8Ygzguax2N3Fa/D/ag1WqbOprdI+uY6wMUl8/a2G+iag==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": "^12.22.0 || ^14.17.0 || >=16.0.0"
      },
      "funding": {
        "url": "https://opencollective.com/eslint"
      }
    },
    "node_modules/@eslint-community/regexpp": {
      "version": "4.12.1",
      "resolved": "https://registry.npmjs.org/@eslint-community/regexpp/-/regexpp-4.12.1.tgz",
      "integrity": "sha512-CCZCDJuduB9OUkFkY2IgppNZMi2lBQgD2qzwXkEia16cge2pijY/aXi96CJMquDMn3nJdlPV1A5KrJEXwfLNzQ==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": "^12.0.0 || ^14.0.0 || >=16.0.0"
      }
    },
    "node_modules/@eslint/config-array": {
      "version": "0.21.0",
      "resolved": "https://registry.npmjs.org/@eslint/config-array/-/config-array-0.21.0.tgz",
      "integrity": "sha512-ENIdc4iLu0d93HeYirvKmrzshzofPw6VkZRKQGe9Nv46ZnWUzcF1xV01dcvEg/1wXUR61OmmlSfyeyO7EvjLxQ==",
      "dev": true,
      "license": "Apache-2.0",
      "dependencies": {
        "@eslint/object-schema": "^2.1.6",
        "debug": "^4.3.1",
        "minimatch": "^3.1.2"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      }
    },
    "node_modules/@eslint/config-helpers": {
      "version": "0.3.1",
      "resolved": "https://registry.npmjs.org/@eslint/config-helpers/-/config-helpers-0.3.1.tgz",
      "integrity": "sha512-xR93k9WhrDYpXHORXpxVL5oHj3Era7wo6k/Wd8/IsQNnZUTzkGS29lyn3nAT05v6ltUuTFVCCYDEGfy2Or/sPA==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      }
    },
    "node_modules/@eslint/core": {
      "version": "0.15.2",
      "resolved": "https://registry.npmjs.org/@eslint/core/-/core-0.15.2.tgz",
      "integrity": "sha512-78Md3/Rrxh83gCxoUc0EiciuOHsIITzLy53m3d9UyiW8y9Dj2D29FeETqyKA+BRK76tnTp6RXWb3pCay8Oyomg==",
      "dev": true,
      "license": "Apache-2.0",
      "dependencies": {
        "@types/json-schema": "^7.0.15"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      }
    },
    "node_modules/@eslint/eslintrc": {
      "version": "3.3.1",
      "resolved": "https://registry.npmjs.org/@eslint/eslintrc/-/eslintrc-3.3.1.tgz",
      "integrity": "sha512-gtF186CXhIl1p4pJNGZw8Yc6RlshoePRvE0X91oPGb3vZ8pM3qOS9W9NGPat9LziaBV7XrJWGylNQXkGcnM3IQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "ajv": "^6.12.4",
        "debug": "^4.3.2",
        "espree": "^10.0.1",
        "globals": "^14.0.0",
        "ignore": "^5.2.0",
        "import-fresh": "^3.2.1",
        "js-yaml": "^4.1.0",
        "minimatch": "^3.1.2",
        "strip-json-comments": "^3.1.1"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      },
      "funding": {
        "url": "https://opencollective.com/eslint"
      }
    },
    "node_modules/@eslint/eslintrc/node_modules/globals": {
      "version": "14.0.0",
      "resolved": "https://registry.npmjs.org/globals/-/globals-14.0.0.tgz",
      "integrity": "sha512-oahGvuMGQlPw/ivIYBjVSrWAfWLBeku5tpPE2fOPLi+WHffIWbuh2tCjhyQhTBPMf5E9jDEH4FOmTYgYwbKwtQ==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=18"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/@eslint/js": {
      "version": "9.36.0",
      "resolved": "https://registry.npmjs.org/@eslint/js/-/js-9.36.0.tgz",
      "integrity": "sha512-uhCbYtYynH30iZErszX78U+nR3pJU3RHGQ57NXy5QupD4SBVwDeU8TNBy+MjMngc1UyIW9noKqsRqfjQTBU2dw==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      },
      "funding": {
        "url": "https://eslint.org/donate"
      }
    },
    "node_modules/@eslint/object-schema": {
      "version": "2.1.6",
      "resolved": "https://registry.npmjs.org/@eslint/object-schema/-/object-schema-2.1.6.tgz",
      "integrity": "sha512-RBMg5FRL0I0gs51M/guSAj5/e14VQ4tpZnQNWwuDT66P14I43ItmPfIZRhO9fUVIPOAQXU47atlywZ/czoqFPA==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      }
    },
    "node_modules/@eslint/plugin-kit": {
      "version": "0.3.5",
      "resolved": "https://registry.npmjs.org/@eslint/plugin-kit/-/plugin-kit-0.3.5.tgz",
      "integrity": "sha512-Z5kJ+wU3oA7MMIqVR9tyZRtjYPr4OC004Q4Rw7pgOKUOKkJfZ3O24nz3WYfGRpMDNmcOi3TwQOmgm7B7Tpii0w==",
      "dev": true,
      "license": "Apache-2.0",
      "dependencies": {
        "@eslint/core": "^0.15.2",
        "levn": "^0.4.1"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      }
    },
    "node_modules/@humanfs/core": {
      "version": "0.19.1",
      "resolved": "https://registry.npmjs.org/@humanfs/core/-/core-0.19.1.tgz",
      "integrity": "sha512-5DyQ4+1JEUzejeK1JGICcideyfUbGixgS9jNgex5nqkW+cY7WZhxBigmieN5Qnw9ZosSNVC9KQKyb+GUaGyKUA==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": ">=18.18.0"
      }
    },
    "node_modules/@humanfs/node": {
      "version": "0.16.7",
      "resolved": "https://registry.npmjs.org/@humanfs/node/-/node-0.16.7.tgz",
      "integrity": "sha512-/zUx+yOsIrG4Y43Eh2peDeKCxlRt/gET6aHfaKpuq267qXdYDFViVHfMaLyygZOnl0kGWxFIgsBy8QFuTLUXEQ==",
      "dev": true,
      "license": "Apache-2.0",
      "dependencies": {
        "@humanfs/core": "^0.19.1",
        "@humanwhocodes/retry": "^0.4.0"
      },
      "engines": {
        "node": ">=18.18.0"
      }
    },
    "node_modules/@humanwhocodes/module-importer": {
      "version": "1.0.1",
      "resolved": "https://registry.npmjs.org/@humanwhocodes/module-importer/-/module-importer-1.0.1.tgz",
      "integrity": "sha512-bxveV4V8v5Yb4ncFTT3rPSgZBOpCkjfK0y4oVVVJwIuDVBRMDXrPyXRL988i5ap9m9bnyEEjWfm5WkBmtffLfA==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": ">=12.22"
      },
      "funding": {
        "type": "github",
        "url": "https://github.com/sponsors/nzakas"
      }
    },
    "node_modules/@humanwhocodes/retry": {
      "version": "0.4.3",
      "resolved": "https://registry.npmjs.org/@humanwhocodes/retry/-/retry-0.4.3.tgz",
      "integrity": "sha512-bV0Tgo9K4hfPCek+aMAn81RppFKv2ySDQeMoSZuvTASywNTnVJCArCZE2FWqpvIatKu7VMRLWlR1EazvVhDyhQ==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": ">=18.18"
      },
      "funding": {
        "type": "github",
        "url": "https://github.com/sponsors/nzakas"
      }
    },
    "node_modules/@isaacs/fs-minipass": {
      "version": "4.0.1",
      "resolved": "https://registry.npmjs.org/@isaacs/fs-minipass/-/fs-minipass-4.0.1.tgz",
      "integrity": "sha512-wgm9Ehl2jpeqP3zw/7mo3kRHFp5MEDhqAdwy1fTGkHAwnkGOVsgpvQhL8B5n1qlb01jV3n/bI0ZfZp5lWA1k4w==",
      "dev": true,
      "license": "ISC",
      "dependencies": {
        "minipass": "^7.0.4"
      },
      "engines": {
        "node": ">=18.0.0"
      }
    },
    "node_modules/@jridgewell/gen-mapping": {
      "version": "0.3.13",
      "resolved": "https://registry.npmjs.org/@jridgewell/gen-mapping/-/gen-mapping-0.3.13.tgz",
      "integrity": "sha512-2kkt/7niJ6MgEPxF0bYdQ6etZaA+fQvDcLKckhy1yIQOzaoKjBBjSj63/aLVjYE3qhRt5dvM+uUyfCg6UKCBbA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@jridgewell/sourcemap-codec": "^1.5.0",
        "@jridgewell/trace-mapping": "^0.3.24"
      }
    },
    "node_modules/@jridgewell/remapping": {
      "version": "2.3.5",
      "resolved": "https://registry.npmjs.org/@jridgewell/remapping/-/remapping-2.3.5.tgz",
      "integrity": "sha512-LI9u/+laYG4Ds1TDKSJW2YPrIlcVYOwi2fUC6xB43lueCjgxV4lffOCZCtYFiH6TNOX+tQKXx97T4IKHbhyHEQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@jridgewell/gen-mapping": "^0.3.5",
        "@jridgewell/trace-mapping": "^0.3.24"
      }
    },
    "node_modules/@jridgewell/resolve-uri": {
      "version": "3.1.2",
      "resolved": "https://registry.npmjs.org/@jridgewell/resolve-uri/-/resolve-uri-3.1.2.tgz",
      "integrity": "sha512-bRISgCIjP20/tbWSPWMEi54QVPRZExkuD9lJL+UIxUKtwVJA8wW1Trb1jMs1RFXo1CBTNZ/5hpC9QvmKWdopKw==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.0.0"
      }
    },
    "node_modules/@jridgewell/sourcemap-codec": {
      "version": "1.5.5",
      "resolved": "https://registry.npmjs.org/@jridgewell/sourcemap-codec/-/sourcemap-codec-1.5.5.tgz",
      "integrity": "sha512-cYQ9310grqxueWbl+WuIUIaiUaDcj7WOq5fVhEljNVgRfOUhY9fy2zTvfoqWsnebh8Sl70VScFbICvJnLKB0Og==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/@jridgewell/trace-mapping": {
      "version": "0.3.31",
      "resolved": "https://registry.npmjs.org/@jridgewell/trace-mapping/-/trace-mapping-0.3.31.tgz",
      "integrity": "sha512-zzNR+SdQSDJzc8joaeP8QQoCQr8NuYx2dIIytl1QeBEZHJ9uW6hebsrYgbz8hJwUQao3TWCMtmfV8Nu1twOLAw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@jridgewell/resolve-uri": "^3.1.0",
        "@jridgewell/sourcemap-codec": "^1.4.14"
      }
    },
    "node_modules/@rolldown/pluginutils": {
      "version": "1.0.0-beta.38",
      "resolved": "https://registry.npmjs.org/@rolldown/pluginutils/-/pluginutils-1.0.0-beta.38.tgz",
      "integrity": "sha512-N/ICGKleNhA5nc9XXQG/kkKHJ7S55u0x0XUJbbkmdCnFuoRkM1Il12q9q0eX19+M7KKUEPw/daUPIRnxhcxAIw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/@rollup/rollup-android-arm-eabi": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm-eabi/-/rollup-android-arm-eabi-4.52.3.tgz",
      "integrity": "sha512-h6cqHGZ6VdnwliFG1NXvMPTy/9PS3h8oLh7ImwR+kl+oYnQizgjxsONmmPSb2C66RksfkfIxEVtDSEcJiO0tqw==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "android"
      ]
    },
    "node_modules/@rollup/rollup-android-arm64": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-android-arm64/-/rollup-android-arm64-4.52.3.tgz",
      "integrity": "sha512-wd+u7SLT/u6knklV/ifG7gr5Qy4GUbH2hMWcDauPFJzmCZUAJ8L2bTkVXC2niOIxp8lk3iH/QX8kSrUxVZrOVw==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "android"
      ]
    },
    "node_modules/@rollup/rollup-darwin-arm64": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-arm64/-/rollup-darwin-arm64-4.52.3.tgz",
      "integrity": "sha512-lj9ViATR1SsqycwFkJCtYfQTheBdvlWJqzqxwc9f2qrcVrQaF/gCuBRTiTolkRWS6KvNxSk4KHZWG7tDktLgjg==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ]
    },
    "node_modules/@rollup/rollup-darwin-x64": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-darwin-x64/-/rollup-darwin-x64-4.52.3.tgz",
      "integrity": "sha512-+Dyo7O1KUmIsbzx1l+4V4tvEVnVQqMOIYtrxK7ncLSknl1xnMHLgn7gddJVrYPNZfEB8CIi3hK8gq8bDhb3h5A==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ]
    },
    "node_modules/@rollup/rollup-freebsd-arm64": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-arm64/-/rollup-freebsd-arm64-4.52.3.tgz",
      "integrity": "sha512-u9Xg2FavYbD30g3DSfNhxgNrxhi6xVG4Y6i9Ur1C7xUuGDW3banRbXj+qgnIrwRN4KeJ396jchwy9bCIzbyBEQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "freebsd"
      ]
    },
    "node_modules/@rollup/rollup-freebsd-x64": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-freebsd-x64/-/rollup-freebsd-x64-4.52.3.tgz",
      "integrity": "sha512-5M8kyi/OX96wtD5qJR89a/3x5x8x5inXBZO04JWhkQb2JWavOWfjgkdvUqibGJeNNaz1/Z1PPza5/tAPXICI6A==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "freebsd"
      ]
    },
    "node_modules/@rollup/rollup-linux-arm-gnueabihf": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-gnueabihf/-/rollup-linux-arm-gnueabihf-4.52.3.tgz",
      "integrity": "sha512-IoerZJ4l1wRMopEHRKOO16e04iXRDyZFZnNZKrWeNquh5d6bucjezgd+OxG03mOMTnS1x7hilzb3uURPkJ0OfA==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-arm-musleabihf": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm-musleabihf/-/rollup-linux-arm-musleabihf-4.52.3.tgz",
      "integrity": "sha512-ZYdtqgHTDfvrJHSh3W22TvjWxwOgc3ThK/XjgcNGP2DIwFIPeAPNsQxrJO5XqleSlgDux2VAoWQ5iJrtaC1TbA==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-arm64-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-gnu/-/rollup-linux-arm64-gnu-4.52.3.tgz",
      "integrity": "sha512-NcViG7A0YtuFDA6xWSgmFb6iPFzHlf5vcqb2p0lGEbT+gjrEEz8nC/EeDHvx6mnGXnGCC1SeVV+8u+smj0CeGQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-arm64-musl": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-arm64-musl/-/rollup-linux-arm64-musl-4.52.3.tgz",
      "integrity": "sha512-d3pY7LWno6SYNXRm6Ebsq0DJGoiLXTb83AIPCXl9fmtIQs/rXoS8SJxxUNtFbJ5MiOvs+7y34np77+9l4nfFMw==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-loong64-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-loong64-gnu/-/rollup-linux-loong64-gnu-4.52.3.tgz",
      "integrity": "sha512-3y5GA0JkBuirLqmjwAKwB0keDlI6JfGYduMlJD/Rl7fvb4Ni8iKdQs1eiunMZJhwDWdCvrcqXRY++VEBbvk6Eg==",
      "cpu": [
        "loong64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-ppc64-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-ppc64-gnu/-/rollup-linux-ppc64-gnu-4.52.3.tgz",
      "integrity": "sha512-AUUH65a0p3Q0Yfm5oD2KVgzTKgwPyp9DSXc3UA7DtxhEb/WSPfbG4wqXeSN62OG5gSo18em4xv6dbfcUGXcagw==",
      "cpu": [
        "ppc64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-riscv64-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-gnu/-/rollup-linux-riscv64-gnu-4.52.3.tgz",
      "integrity": "sha512-1makPhFFVBqZE+XFg3Dkq+IkQ7JvmUrwwqaYBL2CE+ZpxPaqkGaiWFEWVGyvTwZace6WLJHwjVh/+CXbKDGPmg==",
      "cpu": [
        "riscv64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-riscv64-musl": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-riscv64-musl/-/rollup-linux-riscv64-musl-4.52.3.tgz",
      "integrity": "sha512-OOFJa28dxfl8kLOPMUOQBCO6z3X2SAfzIE276fwT52uXDWUS178KWq0pL7d6p1kz7pkzA0yQwtqL0dEPoVcRWg==",
      "cpu": [
        "riscv64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-s390x-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-s390x-gnu/-/rollup-linux-s390x-gnu-4.52.3.tgz",
      "integrity": "sha512-jMdsML2VI5l+V7cKfZx3ak+SLlJ8fKvLJ0Eoa4b9/vCUrzXKgoKxvHqvJ/mkWhFiyp88nCkM5S2v6nIwRtPcgg==",
      "cpu": [
        "s390x"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-x64-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-gnu/-/rollup-linux-x64-gnu-4.52.3.tgz",
      "integrity": "sha512-tPgGd6bY2M2LJTA1uGq8fkSPK8ZLYjDjY+ZLK9WHncCnfIz29LIXIqUgzCR0hIefzy6Hpbe8Th5WOSwTM8E7LA==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-linux-x64-musl": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-linux-x64-musl/-/rollup-linux-x64-musl-4.52.3.tgz",
      "integrity": "sha512-BCFkJjgk+WFzP+tcSMXq77ymAPIxsX9lFJWs+2JzuZTLtksJ2o5hvgTdIcZ5+oKzUDMwI0PfWzRBYAydAHF2Mw==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ]
    },
    "node_modules/@rollup/rollup-openharmony-arm64": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-openharmony-arm64/-/rollup-openharmony-arm64-4.52.3.tgz",
      "integrity": "sha512-KTD/EqjZF3yvRaWUJdD1cW+IQBk4fbQaHYJUmP8N4XoKFZilVL8cobFSTDnjTtxWJQ3JYaMgF4nObY/+nYkumA==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "openharmony"
      ]
    },
    "node_modules/@rollup/rollup-win32-arm64-msvc": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-arm64-msvc/-/rollup-win32-arm64-msvc-4.52.3.tgz",
      "integrity": "sha512-+zteHZdoUYLkyYKObGHieibUFLbttX2r+58l27XZauq0tcWYYuKUwY2wjeCN9oK1Um2YgH2ibd6cnX/wFD7DuA==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ]
    },
    "node_modules/@rollup/rollup-win32-ia32-msvc": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-ia32-msvc/-/rollup-win32-ia32-msvc-4.52.3.tgz",
      "integrity": "sha512-of1iHkTQSo3kr6dTIRX6t81uj/c/b15HXVsPcEElN5sS859qHrOepM5p9G41Hah+CTqSh2r8Bm56dL2z9UQQ7g==",
      "cpu": [
        "ia32"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ]
    },
    "node_modules/@rollup/rollup-win32-x64-gnu": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-gnu/-/rollup-win32-x64-gnu-4.52.3.tgz",
      "integrity": "sha512-s0hybmlHb56mWVZQj8ra9048/WZTPLILKxcvcq+8awSZmyiSUZjjem1AhU3Tf4ZKpYhK4mg36HtHDOe8QJS5PQ==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ]
    },
    "node_modules/@tailwindcss/node": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/node/-/node-4.1.13.tgz",
      "integrity": "sha512-eq3ouolC1oEFOAvOMOBAmfCIqZBJuvWvvYWh5h5iOYfe1HFC6+GZ6EIL0JdM3/niGRJmnrOc+8gl9/HGUaaptw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@jridgewell/remapping": "^2.3.4",
        "enhanced-resolve": "^5.18.3",
        "jiti": "^2.5.1",
        "lightningcss": "1.30.1",
        "magic-string": "^0.30.18",
        "source-map-js": "^1.2.1",
        "tailwindcss": "4.1.13"
      }
    },
    "node_modules/@tailwindcss/oxide": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide/-/oxide-4.1.13.tgz",
      "integrity": "sha512-CPgsM1IpGRa880sMbYmG1s4xhAy3xEt1QULgTJGQmZUeNgXFR7s1YxYygmJyBGtou4SyEosGAGEeYqY7R53bIA==",
      "dev": true,
      "hasInstallScript": true,
      "license": "MIT",
      "dependencies": {
        "detect-libc": "^2.0.4",
        "tar": "^7.4.3"
      },
      "engines": {
        "node": ">= 10"
      },
      "optionalDependencies": {
        "@tailwindcss/oxide-android-arm64": "4.1.13",
        "@tailwindcss/oxide-darwin-arm64": "4.1.13",
        "@tailwindcss/oxide-darwin-x64": "4.1.13",
        "@tailwindcss/oxide-freebsd-x64": "4.1.13",
        "@tailwindcss/oxide-linux-arm-gnueabihf": "4.1.13",
        "@tailwindcss/oxide-linux-arm64-gnu": "4.1.13",
        "@tailwindcss/oxide-linux-arm64-musl": "4.1.13",
        "@tailwindcss/oxide-linux-x64-gnu": "4.1.13",
        "@tailwindcss/oxide-linux-x64-musl": "4.1.13",
        "@tailwindcss/oxide-wasm32-wasi": "4.1.13",
        "@tailwindcss/oxide-win32-arm64-msvc": "4.1.13",
        "@tailwindcss/oxide-win32-x64-msvc": "4.1.13"
      }
    },
    "node_modules/@tailwindcss/oxide-android-arm64": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-android-arm64/-/oxide-android-arm64-4.1.13.tgz",
      "integrity": "sha512-BrpTrVYyejbgGo57yc8ieE+D6VT9GOgnNdmh5Sac6+t0m+v+sKQevpFVpwX3pBrM2qKrQwJ0c5eDbtjouY/+ew==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "android"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-darwin-arm64": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-darwin-arm64/-/oxide-darwin-arm64-4.1.13.tgz",
      "integrity": "sha512-YP+Jksc4U0KHcu76UhRDHq9bx4qtBftp9ShK/7UGfq0wpaP96YVnnjFnj3ZFrUAjc5iECzODl/Ts0AN7ZPOANQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-darwin-x64": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-darwin-x64/-/oxide-darwin-x64-4.1.13.tgz",
      "integrity": "sha512-aAJ3bbwrn/PQHDxCto9sxwQfT30PzyYJFG0u/BWZGeVXi5Hx6uuUOQEI2Fa43qvmUjTRQNZnGqe9t0Zntexeuw==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-freebsd-x64": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-freebsd-x64/-/oxide-freebsd-x64-4.1.13.tgz",
      "integrity": "sha512-Wt8KvASHwSXhKE/dJLCCWcTSVmBj3xhVhp/aF3RpAhGeZ3sVo7+NTfgiN8Vey/Fi8prRClDs6/f0KXPDTZE6nQ==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "freebsd"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-linux-arm-gnueabihf": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-arm-gnueabihf/-/oxide-linux-arm-gnueabihf-4.1.13.tgz",
      "integrity": "sha512-mbVbcAsW3Gkm2MGwA93eLtWrwajz91aXZCNSkGTx/R5eb6KpKD5q8Ueckkh9YNboU8RH7jiv+ol/I7ZyQ9H7Bw==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-linux-arm64-gnu": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-arm64-gnu/-/oxide-linux-arm64-gnu-4.1.13.tgz",
      "integrity": "sha512-wdtfkmpXiwej/yoAkrCP2DNzRXCALq9NVLgLELgLim1QpSfhQM5+ZxQQF8fkOiEpuNoKLp4nKZ6RC4kmeFH0HQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-linux-arm64-musl": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-arm64-musl/-/oxide-linux-arm64-musl-4.1.13.tgz",
      "integrity": "sha512-hZQrmtLdhyqzXHB7mkXfq0IYbxegaqTmfa1p9MBj72WPoDD3oNOh1Lnxf6xZLY9C3OV6qiCYkO1i/LrzEdW2mg==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-linux-x64-gnu": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-x64-gnu/-/oxide-linux-x64-gnu-4.1.13.tgz",
      "integrity": "sha512-uaZTYWxSXyMWDJZNY1Ul7XkJTCBRFZ5Fo6wtjrgBKzZLoJNrG+WderJwAjPzuNZOnmdrVg260DKwXCFtJ/hWRQ==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-linux-x64-musl": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-linux-x64-musl/-/oxide-linux-x64-musl-4.1.13.tgz",
      "integrity": "sha512-oXiPj5mi4Hdn50v5RdnuuIms0PVPI/EG4fxAfFiIKQh5TgQgX7oSuDWntHW7WNIi/yVLAiS+CRGW4RkoGSSgVQ==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-wasm32-wasi": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-wasm32-wasi/-/oxide-wasm32-wasi-4.1.13.tgz",
      "integrity": "sha512-+LC2nNtPovtrDwBc/nqnIKYh/W2+R69FA0hgoeOn64BdCX522u19ryLh3Vf3F8W49XBcMIxSe665kwy21FkhvA==",
      "bundleDependencies": [
        "@napi-rs/wasm-runtime",
        "@emnapi/core",
        "@emnapi/runtime",
        "@tybys/wasm-util",
        "@emnapi/wasi-threads",
        "tslib"
      ],
      "cpu": [
        "wasm32"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "dependencies": {
        "@emnapi/core": "^1.4.5",
        "@emnapi/runtime": "^1.4.5",
        "@emnapi/wasi-threads": "^1.0.4",
        "@napi-rs/wasm-runtime": "^0.2.12",
        "@tybys/wasm-util": "^0.10.0",
        "tslib": "^2.8.0"
      },
      "engines": {
        "node": ">=14.0.0"
      }
    },
    "node_modules/@tailwindcss/oxide-win32-arm64-msvc": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-win32-arm64-msvc/-/oxide-win32-arm64-msvc-4.1.13.tgz",
      "integrity": "sha512-dziTNeQXtoQ2KBXmrjCxsuPk3F3CQ/yb7ZNZNA+UkNTeiTGgfeh+gH5Pi7mRncVgcPD2xgHvkFCh/MhZWSgyQg==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/oxide-win32-x64-msvc": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/oxide-win32-x64-msvc/-/oxide-win32-x64-msvc-4.1.13.tgz",
      "integrity": "sha512-3+LKesjXydTkHk5zXX01b5KMzLV1xl2mcktBJkje7rhFUpUlYJy7IMOLqjIRQncLTa1WZZiFY/foAeB5nmaiTw==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">= 10"
      }
    },
    "node_modules/@tailwindcss/vite": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/@tailwindcss/vite/-/vite-4.1.13.tgz",
      "integrity": "sha512-0PmqLQ010N58SbMTJ7BVJ4I2xopiQn/5i6nlb4JmxzQf8zcS5+m2Cv6tqh+sfDwtIdjoEnOvwsGQ1hkUi8QEHQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@tailwindcss/node": "4.1.13",
        "@tailwindcss/oxide": "4.1.13",
        "tailwindcss": "4.1.13"
      },
      "peerDependencies": {
        "vite": "^5.2.0 || ^6 || ^7"
      }
    },
    "node_modules/@types/babel__core": {
      "version": "7.20.5",
      "resolved": "https://registry.npmjs.org/@types/babel__core/-/babel__core-7.20.5.tgz",
      "integrity": "sha512-qoQprZvz5wQFJwMDqeseRXWv3rqMvhgpbXFfVyWhbx9X47POIA6i/+dXefEmZKoAgOaTdaIgNSMqMIU61yRyzA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/parser": "^7.20.7",
        "@babel/types": "^7.20.7",
        "@types/babel__generator": "*",
        "@types/babel__template": "*",
        "@types/babel__traverse": "*"
      }
    },
    "node_modules/@types/babel__generator": {
      "version": "7.27.0",
      "resolved": "https://registry.npmjs.org/@types/babel__generator/-/babel__generator-7.27.0.tgz",
      "integrity": "sha512-ufFd2Xi92OAVPYsy+P4n7/U7e68fex0+Ee8gSG9KX7eo084CWiQ4sdxktvdl0bOPupXtVJPY19zk6EwWqUQ8lg==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/types": "^7.0.0"
      }
    },
    "node_modules/@types/babel__template": {
      "version": "7.4.4",
      "resolved": "https://registry.npmjs.org/@types/babel__template/-/babel__template-7.4.4.tgz",
      "integrity": "sha512-h/NUaSyG5EyxBIp8YRxo4RMe2/qQgvyowRwVMzhYhBCONbW8PUsg4lkFMrhgZhUe5z3L3MiLDuvyJ/CaPa2A8A==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/parser": "^7.1.0",
        "@babel/types": "^7.0.0"
      }
    },
    "node_modules/@types/babel__traverse": {
      "version": "7.28.0",
      "resolved": "https://registry.npmjs.org/@types/babel__traverse/-/babel__traverse-7.28.0.tgz",
      "integrity": "sha512-8PvcXf70gTDZBgt9ptxJ8elBeBjcLOAcOtoO/mPJjtji1+CdGbHgm77om1GrsPxsiE+uXIpNSK64UYaIwQXd4Q==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/types": "^7.28.2"
      }
    },
    "node_modules/@types/estree": {
      "version": "1.0.8",
      "resolved": "https://registry.npmjs.org/@types/estree/-/estree-1.0.8.tgz",
      "integrity": "sha512-dWHzHa2WqEXI/O1E9OjrocMTKJl2mSrEolh1Iomrv6U+JuNwaHXsXx9bLu5gG7BUWFIN0skIQJQ/L1rIex4X6w==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/@types/json-schema": {
      "version": "7.0.15",
      "resolved": "https://registry.npmjs.org/@types/json-schema/-/json-schema-7.0.15.tgz",
      "integrity": "sha512-5+fP8P8MFNC+AyZCDxrB2pkZFPGzqQWUzpSeuuVLvm8VMcorNYavBqoFcxK8bQz4Qsbn4oUEEem4wDLfcysGHA==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/@types/react": {
      "version": "19.1.15",
      "resolved": "https://registry.npmjs.org/@types/react/-/react-19.1.15.tgz",
      "integrity": "sha512-+kLxJpaJzXybyDyFXYADyP1cznTO8HSuBpenGlnKOAkH4hyNINiywvXS/tGJhsrGGP/gM185RA3xpjY0Yg4erA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "csstype": "^3.0.2"
      }
    },
    "node_modules/@types/react-dom": {
      "version": "19.1.9",
      "resolved": "https://registry.npmjs.org/@types/react-dom/-/react-dom-19.1.9.tgz",
      "integrity": "sha512-qXRuZaOsAdXKFyOhRBg6Lqqc0yay13vN7KrIg4L7N4aaHN68ma9OK3NE1BoDFgFOTfM7zg+3/8+2n8rLUH3OKQ==",
      "dev": true,
      "license": "MIT",
      "peerDependencies": {
        "@types/react": "^19.0.0"
      }
    },
    "node_modules/@vitejs/plugin-react": {
      "version": "5.0.4",
      "resolved": "https://registry.npmjs.org/@vitejs/plugin-react/-/plugin-react-5.0.4.tgz",
      "integrity": "sha512-La0KD0vGkVkSk6K+piWDKRUyg8Rl5iAIKRMH0vMJI0Eg47bq1eOxmoObAaQG37WMW9MSyk7Cs8EIWwJC1PtzKA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@babel/core": "^7.28.4",
        "@babel/plugin-transform-react-jsx-self": "^7.27.1",
        "@babel/plugin-transform-react-jsx-source": "^7.27.1",
        "@rolldown/pluginutils": "1.0.0-beta.38",
        "@types/babel__core": "^7.20.5",
        "react-refresh": "^0.17.0"
      },
      "engines": {
        "node": "^20.19.0 || >=22.12.0"
      },
      "peerDependencies": {
        "vite": "^4.2.0 || ^5.0.0 || ^6.0.0 || ^7.0.0"
      }
    },
    "node_modules/acorn": {
      "version": "8.15.0",
      "resolved": "https://registry.npmjs.org/acorn/-/acorn-8.15.0.tgz",
      "integrity": "sha512-NZyJarBfL7nWwIq+FDL6Zp/yHEhePMNnnJ0y3qfieCrmNvYct8uvtiV41UvlSe6apAfk0fY1FbWx+NwfmpvtTg==",
      "dev": true,
      "license": "MIT",
      "bin": {
        "acorn": "bin/acorn"
      },
      "engines": {
        "node": ">=0.4.0"
      }
    },
    "node_modules/acorn-jsx": {
      "version": "5.3.2",
      "resolved": "https://registry.npmjs.org/acorn-jsx/-/acorn-jsx-5.3.2.tgz",
      "integrity": "sha512-rq9s+JNhf0IChjtDXxllJ7g41oZk5SlXtp0LHwyA5cejwn7vKmKp4pPri6YEePv2PU65sAsegbXtIinmDFDXgQ==",
      "dev": true,
      "license": "MIT",
      "peerDependencies": {
        "acorn": "^6.0.0 || ^7.0.0 || ^8.0.0"
      }
    },
    "node_modules/ajv": {
      "version": "6.12.6",
      "resolved": "https://registry.npmjs.org/ajv/-/ajv-6.12.6.tgz",
      "integrity": "sha512-j3fVLgvTo527anyYyJOGTYJbG+vnnQYvE0m5mmkc1TK+nxAppkCLMIL0aZ4dblVCNoGShhm+kzE4ZUykBoMg4g==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "fast-deep-equal": "^3.1.1",
        "fast-json-stable-stringify": "^2.0.0",
        "json-schema-traverse": "^0.4.1",
        "uri-js": "^4.2.2"
      },
      "funding": {
        "type": "github",
        "url": "https://github.com/sponsors/epoberezkin"
      }
    },
    "node_modules/ansi-styles": {
      "version": "4.3.0",
      "resolved": "https://registry.npmjs.org/ansi-styles/-/ansi-styles-4.3.0.tgz",
      "integrity": "sha512-zbB9rCJAT1rbjiVDb2hqKFHNYLxgtk8NURxZ3IZwD3F6NtxbXZQCnnSi1Lkx+IDohdPlFp222wVALIheZJQSEg==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "color-convert": "^2.0.1"
      },
      "engines": {
        "node": ">=8"
      },
      "funding": {
        "url": "https://github.com/chalk/ansi-styles?sponsor=1"
      }
    },
    "node_modules/argparse": {
      "version": "2.0.1",
      "resolved": "https://registry.npmjs.org/argparse/-/argparse-2.0.1.tgz",
      "integrity": "sha512-8+9WqebbFzpX9OR+Wa6O29asIogeRMzcGtAINdpMHHyAg10f05aSFVBbcEqGf/PXw1EjAZ+q2/bEBg3DvurK3Q==",
      "dev": true,
      "license": "Python-2.0"
    },
    "node_modules/balanced-match": {
      "version": "1.0.2",
      "resolved": "https://registry.npmjs.org/balanced-match/-/balanced-match-1.0.2.tgz",
      "integrity": "sha512-3oSeUO0TMV67hN1AmbXsK4yaqU7tjiHlbxRDZOpH0KW9+CeX4bRAaX0Anxt0tx2MrpRpWwQaPwIlISEJhYU5Pw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/baseline-browser-mapping": {
      "version": "2.8.9",
      "resolved": "https://registry.npmjs.org/baseline-browser-mapping/-/baseline-browser-mapping-2.8.9.tgz",
      "integrity": "sha512-hY/u2lxLrbecMEWSB0IpGzGyDyeoMFQhCvZd2jGFSE5I17Fh01sYUBPCJtkWERw7zrac9+cIghxm/ytJa2X8iA==",
      "dev": true,
      "license": "Apache-2.0",
      "bin": {
        "baseline-browser-mapping": "dist/cli.js"
      }
    },
    "node_modules/brace-expansion": {
      "version": "1.1.12",
      "resolved": "https://registry.npmjs.org/brace-expansion/-/brace-expansion-1.1.12.tgz",
      "integrity": "sha512-9T9UjW3r0UW5c1Q7GTwllptXwhvYmEzFhzMfZ9H7FQWt+uZePjZPjBP/W1ZEyZ1twGWom5/56TF4lPcqjnDHcg==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "balanced-match": "^1.0.0",
        "concat-map": "0.0.1"
      }
    },
    "node_modules/browserslist": {
      "version": "4.26.2",
      "resolved": "https://registry.npmjs.org/browserslist/-/browserslist-4.26.2.tgz",
      "integrity": "sha512-ECFzp6uFOSB+dcZ5BK/IBaGWssbSYBHvuMeMt3MMFyhI0Z8SqGgEkBLARgpRH3hutIgPVsALcMwbDrJqPxQ65A==",
      "dev": true,
      "funding": [
        {
          "type": "opencollective",
          "url": "https://opencollective.com/browserslist"
        },
        {
          "type": "tidelift",
          "url": "https://tidelift.com/funding/github/npm/browserslist"
        },
        {
          "type": "github",
          "url": "https://github.com/sponsors/ai"
        }
      ],
      "license": "MIT",
      "dependencies": {
        "baseline-browser-mapping": "^2.8.3",
        "caniuse-lite": "^1.0.30001741",
        "electron-to-chromium": "^1.5.218",
        "node-releases": "^2.0.21",
        "update-browserslist-db": "^1.1.3"
      },
      "bin": {
        "browserslist": "cli.js"
      },
      "engines": {
        "node": "^6 || ^7 || ^8 || ^9 || ^10 || ^11 || ^12 || >=13.7"
      }
    },
    "node_modules/callsites": {
      "version": "3.1.0",
      "resolved": "https://registry.npmjs.org/callsites/-/callsites-3.1.0.tgz",
      "integrity": "sha512-P8BjAsXvZS+VIDUI11hHCQEv74YT67YUi5JJFNWIqL235sBmjX4+qx9Muvls5ivyNENctx46xQLQ3aTuE7ssaQ==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6"
      }
    },
    "node_modules/caniuse-lite": {
      "version": "1.0.30001745",
      "resolved": "https://registry.npmjs.org/caniuse-lite/-/caniuse-lite-1.0.30001745.tgz",
      "integrity": "sha512-ywt6i8FzvdgrrrGbr1jZVObnVv6adj+0if2/omv9cmR2oiZs30zL4DIyaptKcbOrBdOIc74QTMoJvSE2QHh5UQ==",
      "dev": true,
      "funding": [
        {
          "type": "opencollective",
          "url": "https://opencollective.com/browserslist"
        },
        {
          "type": "tidelift",
          "url": "https://tidelift.com/funding/github/npm/caniuse-lite"
        },
        {
          "type": "github",
          "url": "https://github.com/sponsors/ai"
        }
      ],
      "license": "CC-BY-4.0"
    },
    "node_modules/chalk": {
      "version": "4.1.2",
      "resolved": "https://registry.npmjs.org/chalk/-/chalk-4.1.2.tgz",
      "integrity": "sha512-oKnbhFyRIXpUuez8iBMmyEa4nbj4IOQyuhc/wy9kY7/WVPcwIO9VA668Pu8RkO7+0G76SLROeyw9CpQ061i4mA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "ansi-styles": "^4.1.0",
        "supports-color": "^7.1.0"
      },
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/chalk/chalk?sponsor=1"
      }
    },
    "node_modules/chownr": {
      "version": "3.0.0",
      "resolved": "https://registry.npmjs.org/chownr/-/chownr-3.0.0.tgz",
      "integrity": "sha512-+IxzY9BZOQd/XuYPRmrvEVjF/nqj5kgT4kEq7VofrDoM1MxoRjEWkrCC3EtLi59TVawxTAn+orJwFQcrqEN1+g==",
      "dev": true,
      "license": "BlueOak-1.0.0",
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/color-convert": {
      "version": "2.0.1",
      "resolved": "https://registry.npmjs.org/color-convert/-/color-convert-2.0.1.tgz",
      "integrity": "sha512-RRECPsj7iu/xb5oKYcsFHSppFNnsj/52OVTRKb4zP5onXwVF3zVmmToNcOfGC+CRDpfK/U584fMg38ZHCaElKQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "color-name": "~1.1.4"
      },
      "engines": {
        "node": ">=7.0.0"
      }
    },
    "node_modules/color-name": {
      "version": "1.1.4",
      "resolved": "https://registry.npmjs.org/color-name/-/color-name-1.1.4.tgz",
      "integrity": "sha512-dOy+3AuW3a2wNbZHIuMZpTcgjGuLU/uBL/ubcZF9OXbDo8ff4O8yVp5Bf0efS8uEoYo5q4Fx7dY9OgQGXgAsQA==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/concat-map": {
      "version": "0.0.1",
      "resolved": "https://registry.npmjs.org/concat-map/-/concat-map-0.0.1.tgz",
      "integrity": "sha512-/Srv4dswyQNBfohGpz9o6Yb3Gz3SrUDqBH5rTuhGR7ahtlbYKnVxw2bCFMRljaA7EXHaXZ8wsHdodFvbkhKmqg==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/convert-source-map": {
      "version": "2.0.0",
      "resolved": "https://registry.npmjs.org/convert-source-map/-/convert-source-map-2.0.0.tgz",
      "integrity": "sha512-Kvp459HrV2FEJ1CAsi1Ku+MY3kasH19TFykTz2xWmMeq6bk2NU3XXvfJ+Q61m0xktWwt+1HSYf3JZsTms3aRJg==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/cookie": {
      "version": "1.0.2",
      "resolved": "https://registry.npmjs.org/cookie/-/cookie-1.0.2.tgz",
      "integrity": "sha512-9Kr/j4O16ISv8zBBhJoi4bXOYNTkFLOqSL3UDB0njXxCXNezjeyVrJyGOWtgfs/q2km1gwBcfH8q1yEGoMYunA==",
      "license": "MIT",
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/cross-spawn": {
      "version": "7.0.6",
      "resolved": "https://registry.npmjs.org/cross-spawn/-/cross-spawn-7.0.6.tgz",
      "integrity": "sha512-uV2QOWP2nWzsy2aMp8aRibhi9dlzF5Hgh5SHaB9OiTGEyDTiJJyx0uy51QXdyWbtAHNua4XJzUKca3OzKUd3vA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "path-key": "^3.1.0",
        "shebang-command": "^2.0.0",
        "which": "^2.0.1"
      },
      "engines": {
        "node": ">= 8"
      }
    },
    "node_modules/csstype": {
      "version": "3.1.3",
      "resolved": "https://registry.npmjs.org/csstype/-/csstype-3.1.3.tgz",
      "integrity": "sha512-M1uQkMl8rQK/szD0LNhtqxIPLpimGm8sOBwU7lLnCpSbTyY3yeU1Vc7l4KT5zT4s/yOxHH5O7tIuuLOCnLADRw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/debug": {
      "version": "4.4.3",
      "resolved": "https://registry.npmjs.org/debug/-/debug-4.4.3.tgz",
      "integrity": "sha512-RGwwWnwQvkVfavKVt22FGLw+xYSdzARwm0ru6DhTVA3umU5hZc28V3kO4stgYryrTlLpuvgI9GiijltAjNbcqA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "ms": "^2.1.3"
      },
      "engines": {
        "node": ">=6.0"
      },
      "peerDependenciesMeta": {
        "supports-color": {
          "optional": true
        }
      }
    },
    "node_modules/deep-is": {
      "version": "0.1.4",
      "resolved": "https://registry.npmjs.org/deep-is/-/deep-is-0.1.4.tgz",
      "integrity": "sha512-oIPzksmTg4/MriiaYGO+okXDT7ztn/w3Eptv/+gSIdMdKsJo0u4CfYNFJPy+4SKMuCqGw2wxnA+URMg3t8a/bQ==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/detect-libc": {
      "version": "2.1.1",
      "resolved": "https://registry.npmjs.org/detect-libc/-/detect-libc-2.1.1.tgz",
      "integrity": "sha512-ecqj/sy1jcK1uWrwpR67UhYrIFQ+5WlGxth34WquCbamhFA6hkkwiu37o6J5xCHdo1oixJRfVRw+ywV+Hq/0Aw==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/electron-to-chromium": {
      "version": "1.5.227",
      "resolved": "https://registry.npmjs.org/electron-to-chromium/-/electron-to-chromium-1.5.227.tgz",
      "integrity": "sha512-ITxuoPfJu3lsNWUi2lBM2PaBPYgH3uqmxut5vmBxgYvyI4AlJ6P3Cai1O76mOrkJCBzq0IxWg/NtqOrpu/0gKA==",
      "dev": true,
      "license": "ISC"
    },
    "node_modules/enhanced-resolve": {
      "version": "5.18.3",
      "resolved": "https://registry.npmjs.org/enhanced-resolve/-/enhanced-resolve-5.18.3.tgz",
      "integrity": "sha512-d4lC8xfavMeBjzGr2vECC3fsGXziXZQyJxD868h2M/mBI3PwAuODxAkLkq5HYuvrPYcUtiLzsTo8U3PgX3Ocww==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "graceful-fs": "^4.2.4",
        "tapable": "^2.2.0"
      },
      "engines": {
        "node": ">=10.13.0"
      }
    },
    "node_modules/esbuild": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/esbuild/-/esbuild-0.25.10.tgz",
      "integrity": "sha512-9RiGKvCwaqxO2owP61uQ4BgNborAQskMR6QusfWzQqv7AZOg5oGehdY2pRJMTKuwxd1IDBP4rSbI5lHzU7SMsQ==",
      "dev": true,
      "hasInstallScript": true,
      "license": "MIT",
      "bin": {
        "esbuild": "bin/esbuild"
      },
      "engines": {
        "node": ">=18"

``


``

      },
      "optionalDependencies": {
        "@esbuild/aix-ppc64": "0.25.10",
        "@esbuild/android-arm": "0.25.10",
        "@esbuild/android-arm64": "0.25.10",
        "@esbuild/android-x64": "0.25.10",
        "@esbuild/darwin-arm64": "0.25.10",
        "@esbuild/darwin-x64": "0.25.10",
        "@esbuild/freebsd-arm64": "0.25.10",
        "@esbuild/freebsd-x64": "0.25.10",
        "@esbuild/linux-arm": "0.25.10",
        "@esbuild/linux-arm64": "0.25.10",
        "@esbuild/linux-ia32": "0.25.10",
        "@esbuild/linux-loong64": "0.25.10",
        "@esbuild/linux-mips64el": "0.25.10",
        "@esbuild/linux-ppc64": "0.25.10",
        "@esbuild/linux-riscv64": "0.25.10",
        "@esbuild/linux-s390x": "0.25.10",
        "@esbuild/linux-x64": "0.25.10",
        "@esbuild/netbsd-arm64": "0.25.10",
        "@esbuild/netbsd-x64": "0.25.10",
        "@esbuild/openbsd-arm64": "0.25.10",
        "@esbuild/openbsd-x64": "0.25.10",
        "@esbuild/openharmony-arm64": "0.25.10",
        "@esbuild/sunos-x64": "0.25.10",
        "@esbuild/win32-arm64": "0.25.10",
        "@esbuild/win32-ia32": "0.25.10",
        "@esbuild/win32-x64": "0.25.10"
      }
    },
    "node_modules/esbuild/node_modules/@esbuild/win32-x64": {
      "version": "0.25.10",
      "resolved": "https://registry.npmjs.org/@esbuild/win32-x64/-/win32-x64-0.25.10.tgz",
      "integrity": "sha512-9KpxSVFCu0iK1owoez6aC/s/EdUQLDN3adTxGCqxMVhrPDj6bt5dbrHDXUuq+Bs2vATFBBrQS5vdQ/Ed2P+nbw==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/escalade": {
      "version": "3.2.0",
      "resolved": "https://registry.npmjs.org/escalade/-/escalade-3.2.0.tgz",
      "integrity": "sha512-WUj2qlxaQtO4g6Pq5c29GTcWGDyd8itL8zTlipgECz3JesAiiOKotd8JU6otB3PACgG6xkJUyVhboMS+bje/jA==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6"
      }
    },
    "node_modules/escape-string-regexp": {
      "version": "4.0.0",
      "resolved": "https://registry.npmjs.org/escape-string-regexp/-/escape-string-regexp-4.0.0.tgz",
      "integrity": "sha512-TtpcNJ3XAzx3Gq8sWRzJaVajRs0uVxA2YAkdb1jm2YkPz4G6egUFAyA3n5vtEIZefPk5Wa4UXbKuS5fKkJWdgA==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/eslint": {
      "version": "9.36.0",
      "resolved": "https://registry.npmjs.org/eslint/-/eslint-9.36.0.tgz",
      "integrity": "sha512-hB4FIzXovouYzwzECDcUkJ4OcfOEkXTv2zRY6B9bkwjx/cprAq0uvm1nl7zvQ0/TsUk0zQiN4uPfJpB9m+rPMQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@eslint-community/eslint-utils": "^4.8.0",
        "@eslint-community/regexpp": "^4.12.1",
        "@eslint/config-array": "^0.21.0",
        "@eslint/config-helpers": "^0.3.1",
        "@eslint/core": "^0.15.2",
        "@eslint/eslintrc": "^3.3.1",
        "@eslint/js": "9.36.0",
        "@eslint/plugin-kit": "^0.3.5",
        "@humanfs/node": "^0.16.6",
        "@humanwhocodes/module-importer": "^1.0.1",
        "@humanwhocodes/retry": "^0.4.2",
        "@types/estree": "^1.0.6",
        "@types/json-schema": "^7.0.15",
        "ajv": "^6.12.4",
        "chalk": "^4.0.0",
        "cross-spawn": "^7.0.6",
        "debug": "^4.3.2",
        "escape-string-regexp": "^4.0.0",
        "eslint-scope": "^8.4.0",
        "eslint-visitor-keys": "^4.2.1",
        "espree": "^10.4.0",
        "esquery": "^1.5.0",
        "esutils": "^2.0.2",
        "fast-deep-equal": "^3.1.3",
        "file-entry-cache": "^8.0.0",
        "find-up": "^5.0.0",
        "glob-parent": "^6.0.2",
        "ignore": "^5.2.0",
        "imurmurhash": "^0.1.4",
        "is-glob": "^4.0.0",
        "json-stable-stringify-without-jsonify": "^1.0.1",
        "lodash.merge": "^4.6.2",
        "minimatch": "^3.1.2",
        "natural-compare": "^1.4.0",
        "optionator": "^0.9.3"
      },
      "bin": {
        "eslint": "bin/eslint.js"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      },
      "funding": {
        "url": "https://eslint.org/donate"
      },
      "peerDependencies": {
        "jiti": "*"
      },
      "peerDependenciesMeta": {
        "jiti": {
          "optional": true
        }
      }
    },
    "node_modules/eslint-plugin-react-hooks": {
      "version": "5.2.0",
      "resolved": "https://registry.npmjs.org/eslint-plugin-react-hooks/-/eslint-plugin-react-hooks-5.2.0.tgz",
      "integrity": "sha512-+f15FfK64YQwZdJNELETdn5ibXEUQmW1DZL6KXhNnc2heoy/sg9VJJeT7n8TlMWouzWqSWavFkIhHyIbIAEapg==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=10"
      },
      "peerDependencies": {
        "eslint": "^3.0.0 || ^4.0.0 || ^5.0.0 || ^6.0.0 || ^7.0.0 || ^8.0.0-0 || ^9.0.0"
      }
    },
    "node_modules/eslint-plugin-react-refresh": {
      "version": "0.4.22",
      "resolved": "https://registry.npmjs.org/eslint-plugin-react-refresh/-/eslint-plugin-react-refresh-0.4.22.tgz",
      "integrity": "sha512-atkAG6QaJMGoTLc4MDAP+rqZcfwQuTIh2IqHWFLy2TEjxr0MOK+5BSG4RzL2564AAPpZkDRsZXAUz68kjnU6Ug==",
      "dev": true,
      "license": "MIT",
      "peerDependencies": {
        "eslint": ">=8.40"
      }
    },
    "node_modules/eslint-scope": {
      "version": "8.4.0",
      "resolved": "https://registry.npmjs.org/eslint-scope/-/eslint-scope-8.4.0.tgz",
      "integrity": "sha512-sNXOfKCn74rt8RICKMvJS7XKV/Xk9kA7DyJr8mJik3S7Cwgy3qlkkmyS2uQB3jiJg6VNdZd/pDBJu0nvG2NlTg==",
      "dev": true,
      "license": "BSD-2-Clause",
      "dependencies": {
        "esrecurse": "^4.3.0",
        "estraverse": "^5.2.0"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      },
      "funding": {
        "url": "https://opencollective.com/eslint"
      }
    },
    "node_modules/eslint-visitor-keys": {
      "version": "4.2.1",
      "resolved": "https://registry.npmjs.org/eslint-visitor-keys/-/eslint-visitor-keys-4.2.1.tgz",
      "integrity": "sha512-Uhdk5sfqcee/9H/rCOJikYz67o0a2Tw2hGRPOG2Y1R2dg7brRe1uG0yaNQDHu+TO/uQPF/5eCapvYSmHUjt7JQ==",
      "dev": true,
      "license": "Apache-2.0",
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      },
      "funding": {
        "url": "https://opencollective.com/eslint"
      }
    },
    "node_modules/espree": {
      "version": "10.4.0",
      "resolved": "https://registry.npmjs.org/espree/-/espree-10.4.0.tgz",
      "integrity": "sha512-j6PAQ2uUr79PZhBjP5C5fhl8e39FmRnOjsD5lGnWrFU8i2G776tBK7+nP8KuQUTTyAZUwfQqXAgrVH5MbH9CYQ==",
      "dev": true,
      "license": "BSD-2-Clause",
      "dependencies": {
        "acorn": "^8.15.0",
        "acorn-jsx": "^5.3.2",
        "eslint-visitor-keys": "^4.2.1"
      },
      "engines": {
        "node": "^18.18.0 || ^20.9.0 || >=21.1.0"
      },
      "funding": {
        "url": "https://opencollective.com/eslint"
      }
    },
    "node_modules/esquery": {
      "version": "1.6.0",
      "resolved": "https://registry.npmjs.org/esquery/-/esquery-1.6.0.tgz",
      "integrity": "sha512-ca9pw9fomFcKPvFLXhBKUK90ZvGibiGOvRJNbjljY7s7uq/5YO4BOzcYtJqExdx99rF6aAcnRxHmcUHcz6sQsg==",
      "dev": true,
      "license": "BSD-3-Clause",
      "dependencies": {
        "estraverse": "^5.1.0"
      },
      "engines": {
        "node": ">=0.10"
      }
    },
    "node_modules/esrecurse": {
      "version": "4.3.0",
      "resolved": "https://registry.npmjs.org/esrecurse/-/esrecurse-4.3.0.tgz",
      "integrity": "sha512-KmfKL3b6G+RXvP8N1vr3Tq1kL/oCFgn2NYXEtqP8/L3pKapUA4G8cFVaoF3SU323CD4XypR/ffioHmkti6/Tag==",
      "dev": true,
      "license": "BSD-2-Clause",
      "dependencies": {
        "estraverse": "^5.2.0"
      },
      "engines": {
        "node": ">=4.0"
      }
    },
    "node_modules/estraverse": {
      "version": "5.3.0",
      "resolved": "https://registry.npmjs.org/estraverse/-/estraverse-5.3.0.tgz",
      "integrity": "sha512-MMdARuVEQziNTeJD8DgMqmhwR11BRQ/cBP+pLtYdSTnf3MIO8fFeiINEbX36ZdNlfU/7A9f3gUw49B3oQsvwBA==",
      "dev": true,
      "license": "BSD-2-Clause",
      "engines": {
        "node": ">=4.0"
      }
    },
    "node_modules/esutils": {
      "version": "2.0.3",
      "resolved": "https://registry.npmjs.org/esutils/-/esutils-2.0.3.tgz",
      "integrity": "sha512-kVscqXk4OCp68SZ0dkgEKVi6/8ij300KBWTJq32P/dYeWTSwK41WyTxalN1eRmA5Z9UU/LX9D7FWSmV9SAYx6g==",
      "dev": true,
      "license": "BSD-2-Clause",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/fast-deep-equal": {
      "version": "3.1.3",
      "resolved": "https://registry.npmjs.org/fast-deep-equal/-/fast-deep-equal-3.1.3.tgz",
      "integrity": "sha512-f3qQ9oQy9j2AhBe/H9VC91wLmKBCCU/gDOnKNAYG5hswO7BLKj09Hc5HYNz9cGI++xlpDCIgDaitVs03ATR84Q==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/fast-json-stable-stringify": {
      "version": "2.1.0",
      "resolved": "https://registry.npmjs.org/fast-json-stable-stringify/-/fast-json-stable-stringify-2.1.0.tgz",
      "integrity": "sha512-lhd/wF+Lk98HZoTCtlVraHtfh5XYijIjalXck7saUtuanSDyLMxnHhSXEDJqHxD7msR8D0uCmqlkwjCV8xvwHw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/fast-levenshtein": {
      "version": "2.0.6",
      "resolved": "https://registry.npmjs.org/fast-levenshtein/-/fast-levenshtein-2.0.6.tgz",
      "integrity": "sha512-DCXu6Ifhqcks7TZKY3Hxp3y6qphY5SJZmrWMDrKcERSOXWQdMhU9Ig/PYrzyw/ul9jOIyh0N4M0tbC5hodg8dw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/fdir": {
      "version": "6.5.0",
      "resolved": "https://registry.npmjs.org/fdir/-/fdir-6.5.0.tgz",
      "integrity": "sha512-tIbYtZbucOs0BRGqPJkshJUYdL+SDH7dVM8gjy+ERp3WAUjLEFJE+02kanyHtwjWOnwrKYBiwAmM0p4kLJAnXg==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=12.0.0"
      },
      "peerDependencies": {
        "picomatch": "^3 || ^4"
      },
      "peerDependenciesMeta": {
        "picomatch": {
          "optional": true
        }
      }
    },
    "node_modules/file-entry-cache": {
      "version": "8.0.0",
      "resolved": "https://registry.npmjs.org/file-entry-cache/-/file-entry-cache-8.0.0.tgz",
      "integrity": "sha512-XXTUwCvisa5oacNGRP9SfNtYBNAMi+RPwBFmblZEF7N7swHYQS6/Zfk7SRwx4D5j3CH211YNRco1DEMNVfZCnQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "flat-cache": "^4.0.0"
      },
      "engines": {
        "node": ">=16.0.0"
      }
    },
    "node_modules/find-up": {
      "version": "5.0.0",
      "resolved": "https://registry.npmjs.org/find-up/-/find-up-5.0.0.tgz",
      "integrity": "sha512-78/PXT1wlLLDgTzDs7sjq9hzz0vXD+zn+7wypEe4fXQxCmdmqfGsEPQxmiCSQI3ajFV91bVSsvNtrJRiW6nGng==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "locate-path": "^6.0.0",
        "path-exists": "^4.0.0"
      },
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/flat-cache": {
      "version": "4.0.1",
      "resolved": "https://registry.npmjs.org/flat-cache/-/flat-cache-4.0.1.tgz",
      "integrity": "sha512-f7ccFPK3SXFHpx15UIGyRJ/FJQctuKZ0zVuN3frBo4HnK3cay9VEW0R6yPYFHC0AgqhukPzKjq22t5DmAyqGyw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "flatted": "^3.2.9",
        "keyv": "^4.5.4"
      },
      "engines": {
        "node": ">=16"
      }
    },
    "node_modules/flatted": {
      "version": "3.3.3",
      "resolved": "https://registry.npmjs.org/flatted/-/flatted-3.3.3.tgz",
      "integrity": "sha512-GX+ysw4PBCz0PzosHDepZGANEuFCMLrnRTiEy9McGjmkCQYwRq4A/X786G/fjM/+OjsWSU1ZrY5qyARZmO/uwg==",
      "dev": true,
      "license": "ISC"
    },
    "node_modules/fsevents": {
      "version": "2.3.3",
      "resolved": "https://registry.npmjs.org/fsevents/-/fsevents-2.3.3.tgz",
      "integrity": "sha512-5xoDfX+fL7faATnagmWPpbFtwh/R77WmMMqqHGS65C3vvB0YHrgF+B1YmZ3441tMj5n63k0212XNoJwzlhffQw==",
      "dev": true,
      "hasInstallScript": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": "^8.16.0 || ^10.6.0 || >=11.0.0"
      }
    },
    "node_modules/gensync": {
      "version": "1.0.0-beta.2",
      "resolved": "https://registry.npmjs.org/gensync/-/gensync-1.0.0-beta.2.tgz",
      "integrity": "sha512-3hN7NaskYvMDLQY55gnW3NQ+mesEAepTqlg+VEbj7zzqEMBVNhzcGYYeqFo/TlYz6eQiFcp1HcsCZO+nGgS8zg==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6.9.0"
      }
    },
    "node_modules/glob-parent": {
      "version": "6.0.2",
      "resolved": "https://registry.npmjs.org/glob-parent/-/glob-parent-6.0.2.tgz",
      "integrity": "sha512-XxwI8EOhVQgWp6iDL+3b0r86f4d6AX6zSU55HfB4ydCEuXLXc5FcYeOu+nnGftS4TEju/11rt4KJPTMgbfmv4A==",
      "dev": true,
      "license": "ISC",
      "dependencies": {
        "is-glob": "^4.0.3"
      },
      "engines": {
        "node": ">=10.13.0"
      }
    },
    "node_modules/globals": {
      "version": "16.4.0",
      "resolved": "https://registry.npmjs.org/globals/-/globals-16.4.0.tgz",
      "integrity": "sha512-ob/2LcVVaVGCYN+r14cnwnoDPUufjiYgSqRhiFD0Q1iI4Odora5RE8Iv1D24hAz5oMophRGkGz+yuvQmmUMnMw==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=18"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/graceful-fs": {
      "version": "4.2.11",
      "resolved": "https://registry.npmjs.org/graceful-fs/-/graceful-fs-4.2.11.tgz",
      "integrity": "sha512-RbJ5/jmFcNNCcDV5o9eTnBLJ/HszWV0P73bc+Ff4nS/rJj+YaS6IGyiOL0VoBYX+l1Wrl3k63h/KrH+nhJ0XvQ==",
      "dev": true,
      "license": "ISC"
    },
    "node_modules/has-flag": {
      "version": "4.0.0",
      "resolved": "https://registry.npmjs.org/has-flag/-/has-flag-4.0.0.tgz",
      "integrity": "sha512-EykJT/Q1KjTWctppgIAgfSO0tKVuZUjhgMr17kqTumMl6Afv3EISleU7qZUzoXDFTAHTDC4NOoG/ZxU3EvlMPQ==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/ignore": {
      "version": "5.3.2",
      "resolved": "https://registry.npmjs.org/ignore/-/ignore-5.3.2.tgz",
      "integrity": "sha512-hsBTNUqQTDwkWtcdYI2i06Y/nUBEsNEDJKjWdigLvegy8kDuJAS8uRlpkkcQpyEXL0Z/pjDy5HBmMjRCJ2gq+g==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">= 4"
      }
    },
    "node_modules/import-fresh": {
      "version": "3.3.1",
      "resolved": "https://registry.npmjs.org/import-fresh/-/import-fresh-3.3.1.tgz",
      "integrity": "sha512-TR3KfrTZTYLPB6jUjfx6MF9WcWrHL9su5TObK4ZkYgBdWKPOFoSoQIdEuTuR82pmtxH2spWG9h6etwfr1pLBqQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "parent-module": "^1.0.0",
        "resolve-from": "^4.0.0"
      },
      "engines": {
        "node": ">=6"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/imurmurhash": {
      "version": "0.1.4",
      "resolved": "https://registry.npmjs.org/imurmurhash/-/imurmurhash-0.1.4.tgz",
      "integrity": "sha512-JmXMZ6wuvDmLiHEml9ykzqO6lwFbof0GG4IkcGaENdCRDDmMVnny7s5HsIgHCbaq0w2MyPhDqkhTUgS2LU2PHA==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=0.8.19"
      }
    },
    "node_modules/is-extglob": {
      "version": "2.1.1",
      "resolved": "https://registry.npmjs.org/is-extglob/-/is-extglob-2.1.1.tgz",
      "integrity": "sha512-SbKbANkN603Vi4jEZv49LeVJMn4yGwsbzZworEoyEiutsN3nJYdbO36zfhGJ6QEDpOZIFkDtnq5JRxmvl3jsoQ==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/is-glob": {
      "version": "4.0.3",
      "resolved": "https://registry.npmjs.org/is-glob/-/is-glob-4.0.3.tgz",
      "integrity": "sha512-xelSayHH36ZgE7ZWhli7pW34hNbNl8Ojv5KVmkJD4hBdD3th8Tfk9vYasLM+mXWOZhFkgZfxhLSnrwRr4elSSg==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "is-extglob": "^2.1.1"
      },
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/isexe": {
      "version": "2.0.0",
      "resolved": "https://registry.npmjs.org/isexe/-/isexe-2.0.0.tgz",
      "integrity": "sha512-RHxMLp9lnKHGHRng9QFhRCMbYAcVpn69smSGcq3f36xjgVVWThj4qqLbTLlq7Ssj8B+fIQ1EuCEGI2lKsyQeIw==",
      "dev": true,
      "license": "ISC"
    },
    "node_modules/jiti": {
      "version": "2.6.0",
      "resolved": "https://registry.npmjs.org/jiti/-/jiti-2.6.0.tgz",
      "integrity": "sha512-VXe6RjJkBPj0ohtqaO8vSWP3ZhAKo66fKrFNCll4BTcwljPLz03pCbaNKfzGP5MbrCYcbJ7v0nOYYwUzTEIdXQ==",
      "dev": true,
      "license": "MIT",
      "bin": {
        "jiti": "lib/jiti-cli.mjs"
      }
    },
    "node_modules/js-tokens": {
      "version": "4.0.0",
      "resolved": "https://registry.npmjs.org/js-tokens/-/js-tokens-4.0.0.tgz",
      "integrity": "sha512-RdJUflcE3cUzKiMqQgsCu06FPu9UdIJO0beYbPhHN4k6apgJtifcoCtT9bcxOpYBtpD2kCM6Sbzg4CausW/PKQ==",
      "license": "MIT"
    },
    "node_modules/js-yaml": {
      "version": "4.1.0",
      "resolved": "https://registry.npmjs.org/js-yaml/-/js-yaml-4.1.0.tgz",
      "integrity": "sha512-wpxZs9NoxZaJESJGIZTyDEaYpl0FKSA+FB9aJiyemKhMwkxQg63h4T1KJgUGHpTqPDNRcmmYLugrRjJlBtWvRA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "argparse": "^2.0.1"
      },
      "bin": {
        "js-yaml": "bin/js-yaml.js"
      }
    },
    "node_modules/jsesc": {
      "version": "3.1.0",
      "resolved": "https://registry.npmjs.org/jsesc/-/jsesc-3.1.0.tgz",
      "integrity": "sha512-/sM3dO2FOzXjKQhJuo0Q173wf2KOo8t4I8vHy6lF9poUp7bKT0/NHE8fPX23PwfhnykfqnC2xRxOnVw5XuGIaA==",
      "dev": true,
      "license": "MIT",
      "bin": {
        "jsesc": "bin/jsesc"
      },
      "engines": {
        "node": ">=6"
      }
    },
    "node_modules/json-buffer": {
      "version": "3.0.1",
      "resolved": "https://registry.npmjs.org/json-buffer/-/json-buffer-3.0.1.tgz",
      "integrity": "sha512-4bV5BfR2mqfQTJm+V5tPPdf+ZpuhiIvTuAB5g8kcrXOZpTT/QwwVRWBywX1ozr6lEuPdbHxwaJlm9G6mI2sfSQ==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/json-schema-traverse": {
      "version": "0.4.1",
      "resolved": "https://registry.npmjs.org/json-schema-traverse/-/json-schema-traverse-0.4.1.tgz",
      "integrity": "sha512-xbbCH5dCYU5T8LcEhhuh7HJ88HXuW3qsI3Y0zOZFKfZEHcpWiHU/Jxzk629Brsab/mMiHQti9wMP+845RPe3Vg==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/json-stable-stringify-without-jsonify": {
      "version": "1.0.1",
      "resolved": "https://registry.npmjs.org/json-stable-stringify-without-jsonify/-/json-stable-stringify-without-jsonify-1.0.1.tgz",
      "integrity": "sha512-Bdboy+l7tA3OGW6FjyFHWkP5LuByj1Tk33Ljyq0axyzdk9//JSi2u3fP1QSmd1KNwq6VOKYGlAu87CisVir6Pw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/json5": {
      "version": "2.2.3",
      "resolved": "https://registry.npmjs.org/json5/-/json5-2.2.3.tgz",
      "integrity": "sha512-XmOWe7eyHYH14cLdVPoyg+GOH3rYX++KpzrylJwSW98t3Nk+U8XOl8FWKOgwtzdb8lXGf6zYwDUzeHMWfxasyg==",
      "dev": true,
      "license": "MIT",
      "bin": {
        "json5": "lib/cli.js"
      },
      "engines": {
        "node": ">=6"
      }
    },
    "node_modules/keyv": {
      "version": "4.5.4",
      "resolved": "https://registry.npmjs.org/keyv/-/keyv-4.5.4.tgz",
      "integrity": "sha512-oxVHkHR/EJf2CNXnWxRLW6mg7JyCCUcG0DtEGmL2ctUo1PNTin1PUil+r/+4r5MpVgC/fn1kjsx7mjSujKqIpw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "json-buffer": "3.0.1"
      }
    },
    "node_modules/levn": {
      "version": "0.4.1",
      "resolved": "https://registry.npmjs.org/levn/-/levn-0.4.1.tgz",
      "integrity": "sha512-+bT2uH4E5LGE7h/n3evcS/sQlJXCpIp6ym8OWJ5eV6+67Dsql/LaaT7qJBAt2rzfoa/5QBGBhxDix1dMt2kQKQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "prelude-ls": "^1.2.1",
        "type-check": "~0.4.0"
      },
      "engines": {
        "node": ">= 0.8.0"
      }
    },
    "node_modules/lightningcss": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss/-/lightningcss-1.30.1.tgz",
      "integrity": "sha512-xi6IyHML+c9+Q3W0S4fCQJOym42pyurFiJUHEcEyHS0CeKzia4yZDEsLlqOFykxOdHpNy0NmvVO31vcSqAxJCg==",
      "dev": true,
      "license": "MPL-2.0",
      "dependencies": {
        "detect-libc": "^2.0.3"
      },
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      },
      "optionalDependencies": {
        "lightningcss-darwin-arm64": "1.30.1",
        "lightningcss-darwin-x64": "1.30.1",
        "lightningcss-freebsd-x64": "1.30.1",
        "lightningcss-linux-arm-gnueabihf": "1.30.1",
        "lightningcss-linux-arm64-gnu": "1.30.1",
        "lightningcss-linux-arm64-musl": "1.30.1",
        "lightningcss-linux-x64-gnu": "1.30.1",
        "lightningcss-linux-x64-musl": "1.30.1",
        "lightningcss-win32-arm64-msvc": "1.30.1",
        "lightningcss-win32-x64-msvc": "1.30.1"
      }
    },
    "node_modules/lightningcss-darwin-arm64": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-darwin-arm64/-/lightningcss-darwin-arm64-1.30.1.tgz",
      "integrity": "sha512-c8JK7hyE65X1MHMN+Viq9n11RRC7hgin3HhYKhrMyaXflk5GVplZ60IxyoVtzILeKr+xAJwg6zK6sjTBJ0FKYQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-darwin-x64": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-darwin-x64/-/lightningcss-darwin-x64-1.30.1.tgz",
      "integrity": "sha512-k1EvjakfumAQoTfcXUcHQZhSpLlkAuEkdMBsI/ivWw9hL+7FtilQc0Cy3hrx0AAQrVtQAbMI7YjCgYgvn37PzA==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "darwin"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-freebsd-x64": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-freebsd-x64/-/lightningcss-freebsd-x64-1.30.1.tgz",
      "integrity": "sha512-kmW6UGCGg2PcyUE59K5r0kWfKPAVy4SltVeut+umLCFoJ53RdCUWxcRDzO1eTaxf/7Q2H7LTquFHPL5R+Gjyig==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "freebsd"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-linux-arm-gnueabihf": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-linux-arm-gnueabihf/-/lightningcss-linux-arm-gnueabihf-1.30.1.tgz",
      "integrity": "sha512-MjxUShl1v8pit+6D/zSPq9S9dQ2NPFSQwGvxBCYaBYLPlCWuPh9/t1MRS8iUaR8i+a6w7aps+B4N0S1TYP/R+Q==",
      "cpu": [
        "arm"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-linux-arm64-gnu": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-linux-arm64-gnu/-/lightningcss-linux-arm64-gnu-1.30.1.tgz",
      "integrity": "sha512-gB72maP8rmrKsnKYy8XUuXi/4OctJiuQjcuqWNlJQ6jZiWqtPvqFziskH3hnajfvKB27ynbVCucKSm2rkQp4Bw==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-linux-arm64-musl": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-linux-arm64-musl/-/lightningcss-linux-arm64-musl-1.30.1.tgz",
      "integrity": "sha512-jmUQVx4331m6LIX+0wUhBbmMX7TCfjF5FoOH6SD1CttzuYlGNVpA7QnrmLxrsub43ClTINfGSYyHe2HWeLl5CQ==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-linux-x64-gnu": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-linux-x64-gnu/-/lightningcss-linux-x64-gnu-1.30.1.tgz",
      "integrity": "sha512-piWx3z4wN8J8z3+O5kO74+yr6ze/dKmPnI7vLqfSqI8bccaTGY5xiSGVIJBDd5K5BHlvVLpUB3S2YCfelyJ1bw==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-linux-x64-musl": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-linux-x64-musl/-/lightningcss-linux-x64-musl-1.30.1.tgz",
      "integrity": "sha512-rRomAK7eIkL+tHY0YPxbc5Dra2gXlI63HL+v1Pdi1a3sC+tJTcFrHX+E86sulgAXeI7rSzDYhPSeHHjqFhqfeQ==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "linux"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-win32-arm64-msvc": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-win32-arm64-msvc/-/lightningcss-win32-arm64-msvc-1.30.1.tgz",
      "integrity": "sha512-mSL4rqPi4iXq5YVqzSsJgMVFENoa4nGTT/GjO2c0Yl9OuQfPsIfncvLrEW6RbbB24WtZ3xP/2CCmI3tNkNV4oA==",
      "cpu": [
        "arm64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/lightningcss-win32-x64-msvc": {
      "version": "1.30.1",
      "resolved": "https://registry.npmjs.org/lightningcss-win32-x64-msvc/-/lightningcss-win32-x64-msvc-1.30.1.tgz",
      "integrity": "sha512-PVqXh48wh4T53F/1CCu8PIPCxLzWyCnn/9T5W1Jpmdy5h9Cwd+0YQS6/LwhHXSafuc61/xg9Lv5OrCby6a++jg==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MPL-2.0",
      "optional": true,
      "os": [
        "win32"
      ],
      "engines": {
        "node": ">= 12.0.0"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/parcel"
      }
    },
    "node_modules/locate-path": {
      "version": "6.0.0",
      "resolved": "https://registry.npmjs.org/locate-path/-/locate-path-6.0.0.tgz",
      "integrity": "sha512-iPZK6eYjbxRu3uB4/WZ3EsEIMJFMqAoopl3R+zuq0UjcAm/MO6KCweDgPfP3elTztoKP3KtnVHxTn2NHBSDVUw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "p-locate": "^5.0.0"
      },
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/lodash.merge": {
      "version": "4.6.2",
      "resolved": "https://registry.npmjs.org/lodash.merge/-/lodash.merge-4.6.2.tgz",
      "integrity": "sha512-0KpjqXRVvrYyCsX1swR/XTK0va6VQkQM6MNo7PqW77ByjAhoARA8EfrP1N4+KlKj8YS0ZUCtRT/YUuhyYDujIQ==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/loose-envify": {
      "version": "1.4.0",
      "resolved": "https://registry.npmjs.org/loose-envify/-/loose-envify-1.4.0.tgz",
      "integrity": "sha512-lyuxPGr/Wfhrlem2CL/UcnUc1zcqKAImBDzukY7Y5F/yQiNdko6+fRLevlw1HgMySw7f611UIY408EtxRSoK3Q==",
      "license": "MIT",
      "dependencies": {
        "js-tokens": "^3.0.0 || ^4.0.0"
      },
      "bin": {
        "loose-envify": "cli.js"
      }
    },
    "node_modules/lru-cache": {
      "version": "5.1.1",
      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-5.1.1.tgz",
      "integrity": "sha512-KpNARQA3Iwv+jTA0utUVVbrh+Jlrr1Fv0e56GGzAFOXN7dk/FviaDW8LHmK52DlcH4WP2n6gI8vN1aesBFgo9w==",
      "dev": true,
      "license": "ISC",
      "dependencies": {
        "yallist": "^3.0.2"
      }
    },
    "node_modules/magic-string": {
      "version": "0.30.19",
      "resolved": "https://registry.npmjs.org/magic-string/-/magic-string-0.30.19.tgz",
      "integrity": "sha512-2N21sPY9Ws53PZvsEpVtNuSW+ScYbQdp4b9qUaL+9QkHUrGFKo56Lg9Emg5s9V/qrtNBmiR01sYhUOwu3H+VOw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@jridgewell/sourcemap-codec": "^1.5.5"
      }
    },
    "node_modules/minimatch": {
      "version": "3.1.2",
      "resolved": "https://registry.npmjs.org/minimatch/-/minimatch-3.1.2.tgz",
      "integrity": "sha512-J7p63hRiAjw1NDEww1W7i37+ByIrOWO5XQQAzZ3VOcL0PNybwpfmV/N05zFAzwQ9USyEcX6t3UO+K5aqBQOIHw==",
      "dev": true,
      "license": "ISC",
      "dependencies": {
        "brace-expansion": "^1.1.7"
      },
      "engines": {
        "node": "*"
      }
    },
    "node_modules/minipass": {
      "version": "7.1.2",
      "resolved": "https://registry.npmjs.org/minipass/-/minipass-7.1.2.tgz",
      "integrity": "sha512-qOOzS1cBTWYF4BH8fVePDBOO9iptMnGUEZwNc/cMWnTV2nVLZ7VoNWEPHkYczZA0pdoA7dl6e7FL659nX9S2aw==",
      "dev": true,
      "license": "ISC",
      "engines": {
        "node": ">=16 || 14 >=14.17"
      }
    },
    "node_modules/minizlib": {
      "version": "3.1.0",
      "resolved": "https://registry.npmjs.org/minizlib/-/minizlib-3.1.0.tgz",
      "integrity": "sha512-KZxYo1BUkWD2TVFLr0MQoM8vUUigWD3LlD83a/75BqC+4qE0Hb1Vo5v1FgcfaNXvfXzr+5EhQ6ing/CaBijTlw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "minipass": "^7.1.2"
      },
      "engines": {
        "node": ">= 18"
      }
    },
    "node_modules/ms": {
      "version": "2.1.3",
      "resolved": "https://registry.npmjs.org/ms/-/ms-2.1.3.tgz",
      "integrity": "sha512-6FlzubTLZG3J2a/NVCAleEhjzq5oxgHyaCU9yYXvcLsvoVaHJq/s5xXI6/XXP6tz7R9xAOtHnSO/tXtF3WRTlA==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/nanoid": {
      "version": "3.3.11",
      "resolved": "https://registry.npmjs.org/nanoid/-/nanoid-3.3.11.tgz",
      "integrity": "sha512-N8SpfPUnUp1bK+PMYW8qSWdl9U+wwNWI4QKxOYDy9JAro3WMX7p2OeVRF9v+347pnakNevPmiHhNmZ2HbFA76w==",
      "dev": true,
      "funding": [
        {
          "type": "github",
          "url": "https://github.com/sponsors/ai"
        }
      ],
      "license": "MIT",
      "bin": {
        "nanoid": "bin/nanoid.cjs"
      },
      "engines": {
        "node": "^10 || ^12 || ^13.7 || ^14 || >=15.0.1"
      }
    },
    "node_modules/natural-compare": {
      "version": "1.4.0",
      "resolved": "https://registry.npmjs.org/natural-compare/-/natural-compare-1.4.0.tgz",
      "integrity": "sha512-OWND8ei3VtNC9h7V60qff3SVobHr996CTwgxubgyQYEpg290h9J0buyECNNJexkFm5sOajh5G116RYA1c8ZMSw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/node-releases": {
      "version": "2.0.21",
      "resolved": "https://registry.npmjs.org/node-releases/-/node-releases-2.0.21.tgz",
      "integrity": "sha512-5b0pgg78U3hwXkCM8Z9b2FJdPZlr9Psr9V2gQPESdGHqbntyFJKFW4r5TeWGFzafGY3hzs1JC62VEQMbl1JFkw==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/object-assign": {
      "version": "4.1.1",
      "resolved": "https://registry.npmjs.org/object-assign/-/object-assign-4.1.1.tgz",
      "integrity": "sha512-rJgTQnkUnH1sFw8yT6VSU3zD3sWmu6sZhIseY8VX+GRu3P6F7Fu+JNDoXfklElbLJSnc3FUQHVe4cU5hj+BcUg==",
      "license": "MIT",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/optionator": {
      "version": "0.9.4",
      "resolved": "https://registry.npmjs.org/optionator/-/optionator-0.9.4.tgz",
      "integrity": "sha512-6IpQ7mKUxRcZNLIObR0hz7lxsapSSIYNZJwXPGeF0mTVqGKFIXj1DQcMoT22S3ROcLyY/rz0PWaWZ9ayWmad9g==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "deep-is": "^0.1.3",
        "fast-levenshtein": "^2.0.6",
        "levn": "^0.4.1",
        "prelude-ls": "^1.2.1",
        "type-check": "^0.4.0",
        "word-wrap": "^1.2.5"
      },
      "engines": {
        "node": ">= 0.8.0"
      }
    },
    "node_modules/p-limit": {
      "version": "3.1.0",
      "resolved": "https://registry.npmjs.org/p-limit/-/p-limit-3.1.0.tgz",
      "integrity": "sha512-TYOanM3wGwNGsZN2cVTYPArw454xnXj5qmWF1bEoAc4+cU/ol7GVh7odevjp1FNHduHc3KZMcFduxU5Xc6uJRQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "yocto-queue": "^0.1.0"
      },
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/p-locate": {
      "version": "5.0.0",
      "resolved": "https://registry.npmjs.org/p-locate/-/p-locate-5.0.0.tgz",
      "integrity": "sha512-LaNjtRWUBY++zB5nE/NwcaoMylSPk+S+ZHNB1TzdbMJMny6dynpAGt7X/tl/QYq3TIeE6nxHppbo2LGymrG5Pw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "p-limit": "^3.0.2"
      },
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/parent-module": {
      "version": "1.0.1",
      "resolved": "https://registry.npmjs.org/parent-module/-/parent-module-1.0.1.tgz",
      "integrity": "sha512-GQ2EWRpQV8/o+Aw8YqtfZZPfNRWZYkbidE9k5rpl/hC3vtHHBfGm2Ifi6qWV+coDGkrUKZAxE3Lot5kcsRlh+g==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "callsites": "^3.0.0"
      },
      "engines": {
        "node": ">=6"
      }
    },
    "node_modules/path-exists": {
      "version": "4.0.0",
      "resolved": "https://registry.npmjs.org/path-exists/-/path-exists-4.0.0.tgz",
      "integrity": "sha512-ak9Qy5Q7jYb2Wwcey5Fpvg2KoAc/ZIhLSLOSBmRmygPsGwkVVt0fZa0qrtMz+m6tJTAHfZQ8FnmB4MG4LWy7/w==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/path-key": {
      "version": "3.1.1",
      "resolved": "https://registry.npmjs.org/path-key/-/path-key-3.1.1.tgz",
      "integrity": "sha512-ojmeN0qd+y0jszEtoY48r0Peq5dwMEkIlCOu6Q5f41lfkswXuKtYrhgoTpLnyIcHm24Uhqx+5Tqm2InSwLhE6Q==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/picocolors": {
      "version": "1.1.1",
      "resolved": "https://registry.npmjs.org/picocolors/-/picocolors-1.1.1.tgz",
      "integrity": "sha512-xceH2snhtb5M9liqDsmEw56le376mTZkEX/jEb/RxNFyegNul7eNslCXP9FDj/Lcu0X8KEyMceP2ntpaHrDEVA==",
      "dev": true,
      "license": "ISC"
    },
    "node_modules/picomatch": {
      "version": "4.0.3",
      "resolved": "https://registry.npmjs.org/picomatch/-/picomatch-4.0.3.tgz",
      "integrity": "sha512-5gTmgEY/sqK6gFXLIsQNH19lWb4ebPDLA4SdLP7dsWkIXHWlG66oPuVvXSGFPppYZz8ZDZq0dYYrbHfBCVUb1Q==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=12"
      },
      "funding": {
        "url": "https://github.com/sponsors/jonschlinkert"
      }
    },
    "node_modules/postcss": {
      "version": "8.5.6",
      "resolved": "https://registry.npmjs.org/postcss/-/postcss-8.5.6.tgz",
      "integrity": "sha512-3Ybi1tAuwAP9s0r1UQ2J4n5Y0G05bJkpUIO0/bI9MhwmD70S5aTWbXGBwxHrelT+XM1k6dM0pk+SwNkpTRN7Pg==",
      "dev": true,
      "funding": [
        {
          "type": "opencollective",
          "url": "https://opencollective.com/postcss/"
        },
        {
          "type": "tidelift",
          "url": "https://tidelift.com/funding/github/npm/postcss"
        },
        {
          "type": "github",
          "url": "https://github.com/sponsors/ai"
        }
      ],
      "license": "MIT",
      "dependencies": {
        "nanoid": "^3.3.11",
        "picocolors": "^1.1.1",
        "source-map-js": "^1.2.1"
      },
      "engines": {
        "node": "^10 || ^12 || >=14"
      }
    },
    "node_modules/prelude-ls": {
      "version": "1.2.1",
      "resolved": "https://registry.npmjs.org/prelude-ls/-/prelude-ls-1.2.1.tgz",
      "integrity": "sha512-vkcDPrRZo1QZLbn5RLGPpg/WmIQ65qoWWhcGKf/b5eplkkarX0m9z8ppCat4mlOqUsWpyNuYgO3VRyrYHSzX5g==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">= 0.8.0"
      }
    },
    "node_modules/prop-types": {
      "version": "15.8.1",
      "resolved": "https://registry.npmjs.org/prop-types/-/prop-types-15.8.1.tgz",
      "integrity": "sha512-oj87CgZICdulUohogVAR7AjlC0327U4el4L6eAvOqCeudMDVU0NThNaV+b9Df4dXgSP1gXMTnPdhfe/2qDH5cg==",
      "license": "MIT",
      "dependencies": {
        "loose-envify": "^1.4.0",
        "object-assign": "^4.1.1",
        "react-is": "^16.13.1"
      }
    },
    "node_modules/punycode": {
      "version": "2.3.1",
      "resolved": "https://registry.npmjs.org/punycode/-/punycode-2.3.1.tgz",
      "integrity": "sha512-vYt7UD1U9Wg6138shLtLOvdAu+8DsC/ilFtEVHcH+wydcSpNE20AfSOduf6MkRFahL5FY7X1oU7nKVZFtfq8Fg==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6"
      }
    },
    "node_modules/react": {
      "version": "19.1.1",
      "resolved": "https://registry.npmjs.org/react/-/react-19.1.1.tgz",
      "integrity": "sha512-w8nqGImo45dmMIfljjMwOGtbmC/mk4CMYhWIicdSflH91J9TyCyczcPFXJzrZ/ZXcgGRFeP6BU0BEJTw6tZdfQ==",
      "license": "MIT",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/react-bootstrap-icons": {
      "version": "1.11.6",
      "resolved": "https://registry.npmjs.org/react-bootstrap-icons/-/react-bootstrap-icons-1.11.6.tgz",
      "integrity": "sha512-ycXiyeSyzbS1C4+MlPTYe0riB+UlZ7LV7YZQYqlERV2cxDiKtntI0huHmP/3VVvzPt4tGxqK0K+Y6g7We3U6tQ==",
      "license": "MIT",
      "dependencies": {
        "prop-types": "^15.7.2"
      },
      "peerDependencies": {
        "react": ">=16.8.6"
      }
    },
    "node_modules/react-dom": {
      "version": "19.1.1",
      "resolved": "https://registry.npmjs.org/react-dom/-/react-dom-19.1.1.tgz",
      "integrity": "sha512-Dlq/5LAZgF0Gaz6yiqZCf6VCcZs1ghAJyrsu84Q/GT0gV+mCxbfmKNoGRKBYMJ8IEdGPqu49YWXD02GCknEDkw==",
      "license": "MIT",
      "dependencies": {
        "scheduler": "^0.26.0"
      },
      "peerDependencies": {
        "react": "^19.1.1"
      }
    },
    "node_modules/react-is": {
      "version": "16.13.1",
      "resolved": "https://registry.npmjs.org/react-is/-/react-is-16.13.1.tgz",
      "integrity": "sha512-24e6ynE2H+OKt4kqsOvNd8kBpV65zoxbA4BVsEOB3ARVWQki/DHzaUoC5KuON/BiccDaCCTZBuOcfZs70kR8bQ==",
      "license": "MIT"
    },
    "node_modules/react-refresh": {
      "version": "0.17.0",
      "resolved": "https://registry.npmjs.org/react-refresh/-/react-refresh-0.17.0.tgz",
      "integrity": "sha512-z6F7K9bV85EfseRCp2bzrpyQ0Gkw1uLoCel9XBVWPg/TjRj94SkJzUTGfOa4bs7iJvBWtQG0Wq7wnI0syw3EBQ==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/react-router": {
      "version": "7.9.3",
      "resolved": "https://registry.npmjs.org/react-router/-/react-router-7.9.3.tgz",
      "integrity": "sha512-4o2iWCFIwhI/eYAIL43+cjORXYn/aRQPgtFRRZb3VzoyQ5Uej0Bmqj7437L97N9NJW4wnicSwLOLS+yCXfAPgg==",
      "license": "MIT",
      "dependencies": {
        "cookie": "^1.0.1",
        "set-cookie-parser": "^2.6.0"
      },
      "engines": {
        "node": ">=20.0.0"
      },
      "peerDependencies": {
        "react": ">=18",
        "react-dom": ">=18"
      },
      "peerDependenciesMeta": {
        "react-dom": {
          "optional": true
        }
      }
    },
    "node_modules/react-router-dom": {
      "version": "7.9.3",
      "resolved": "https://registry.npmjs.org/react-router-dom/-/react-router-dom-7.9.3.tgz",
      "integrity": "sha512-1QSbA0TGGFKTAc/aWjpfW/zoEukYfU4dc1dLkT/vvf54JoGMkW+fNA+3oyo2gWVW1GM7BxjJVHz5GnPJv40rvg==",
      "license": "MIT",
      "dependencies": {
        "react-router": "7.9.3"
      },
      "engines": {
        "node": ">=20.0.0"
      },
      "peerDependencies": {
        "react": ">=18",
        "react-dom": ">=18"
      }
    },
    "node_modules/resolve-from": {
      "version": "4.0.0",
      "resolved": "https://registry.npmjs.org/resolve-from/-/resolve-from-4.0.0.tgz",
      "integrity": "sha512-pb/MYmXstAkysRFx8piNI1tGFNQIFA3vkE3Gq4EuA1dF6gHp/+vgZqsCGJapvy8N3Q+4o7FwvquPJcnZ7RYy4g==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=4"
      }
    },
    "node_modules/rollup": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/rollup/-/rollup-4.52.3.tgz",
      "integrity": "sha512-RIDh866U8agLgiIcdpB+COKnlCreHJLfIhWC3LVflku5YHfpnsIKigRZeFfMfCc4dVcqNVfQQ5gO/afOck064A==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "@types/estree": "1.0.8"
      },
      "bin": {
        "rollup": "dist/bin/rollup"
      },
      "engines": {
        "node": ">=18.0.0",
        "npm": ">=8.0.0"
      },
      "optionalDependencies": {
        "@rollup/rollup-android-arm-eabi": "4.52.3",
        "@rollup/rollup-android-arm64": "4.52.3",
        "@rollup/rollup-darwin-arm64": "4.52.3",
        "@rollup/rollup-darwin-x64": "4.52.3",
        "@rollup/rollup-freebsd-arm64": "4.52.3",
        "@rollup/rollup-freebsd-x64": "4.52.3",
        "@rollup/rollup-linux-arm-gnueabihf": "4.52.3",
        "@rollup/rollup-linux-arm-musleabihf": "4.52.3",
        "@rollup/rollup-linux-arm64-gnu": "4.52.3",
        "@rollup/rollup-linux-arm64-musl": "4.52.3",
        "@rollup/rollup-linux-loong64-gnu": "4.52.3",
        "@rollup/rollup-linux-ppc64-gnu": "4.52.3",
        "@rollup/rollup-linux-riscv64-gnu": "4.52.3",
        "@rollup/rollup-linux-riscv64-musl": "4.52.3",
        "@rollup/rollup-linux-s390x-gnu": "4.52.3",
        "@rollup/rollup-linux-x64-gnu": "4.52.3",
        "@rollup/rollup-linux-x64-musl": "4.52.3",
        "@rollup/rollup-openharmony-arm64": "4.52.3",
        "@rollup/rollup-win32-arm64-msvc": "4.52.3",
        "@rollup/rollup-win32-ia32-msvc": "4.52.3",
        "@rollup/rollup-win32-x64-gnu": "4.52.3",
        "@rollup/rollup-win32-x64-msvc": "4.52.3",
        "fsevents": "~2.3.2"
      }
    },
    "node_modules/rollup/node_modules/@rollup/rollup-win32-x64-msvc": {
      "version": "4.52.3",
      "resolved": "https://registry.npmjs.org/@rollup/rollup-win32-x64-msvc/-/rollup-win32-x64-msvc-4.52.3.tgz",
      "integrity": "sha512-zGIbEVVXVtauFgl3MRwGWEN36P5ZGenHRMgNw88X5wEhEBpq0XrMEZwOn07+ICrwM17XO5xfMZqh0OldCH5VTA==",
      "cpu": [
        "x64"
      ],
      "dev": true,
      "license": "MIT",
      "optional": true,
      "os": [
        "win32"
      ]
    },
    "node_modules/scheduler": {
      "version": "0.26.0",
      "resolved": "https://registry.npmjs.org/scheduler/-/scheduler-0.26.0.tgz",
      "integrity": "sha512-NlHwttCI/l5gCPR3D1nNXtWABUmBwvZpEQiD4IXSbIDq8BzLIK/7Ir5gTFSGZDUu37K5cMNp0hFtzO38sC7gWA==",
      "license": "MIT"
    },
    "node_modules/semver": {
      "version": "6.3.1",
      "resolved": "https://registry.npmjs.org/semver/-/semver-6.3.1.tgz",
      "integrity": "sha512-BR7VvDCVHO+q2xBEWskxS6DJE1qRnb7DxzUrogb71CWoSficBxYsiAGd+Kl0mmq/MprG9yArRkyrQxTO6XjMzA==",
      "dev": true,
      "license": "ISC",
      "bin": {
        "semver": "bin/semver.js"
      }
    },
    "node_modules/set-cookie-parser": {
      "version": "2.7.1",
      "resolved": "https://registry.npmjs.org/set-cookie-parser/-/set-cookie-parser-2.7.1.tgz",
      "integrity": "sha512-IOc8uWeOZgnb3ptbCURJWNjWUPcO3ZnTTdzsurqERrP6nPyv+paC55vJM0LpOlT2ne+Ix+9+CRG1MNLlyZ4GjQ==",
      "license": "MIT"
    },
    "node_modules/shebang-command": {
      "version": "2.0.0",
      "resolved": "https://registry.npmjs.org/shebang-command/-/shebang-command-2.0.0.tgz",
      "integrity": "sha512-kHxr2zZpYtdmrN1qDjrrX/Z1rR1kG8Dx+gkpK1G4eXmvXswmcE1hTWBWYUzlraYw1/yZp6YuDY77YtvbN0dmDA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "shebang-regex": "^3.0.0"
      },
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/shebang-regex": {
      "version": "3.0.0",
      "resolved": "https://registry.npmjs.org/shebang-regex/-/shebang-regex-3.0.0.tgz",
      "integrity": "sha512-7++dFhtcx3353uBaq8DDR4NuxBetBzC7ZQOhmTQInHEd6bSrXdiEyzCvG07Z44UYdLShWUyXt5M/yhz8ekcb1A==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/source-map-js": {
      "version": "1.2.1",
      "resolved": "https://registry.npmjs.org/source-map-js/-/source-map-js-1.2.1.tgz",
      "integrity": "sha512-UXWMKhLOwVKb728IUtQPXxfYU+usdybtUrK/8uGE8CQMvrhOpwvzDBwj0QhSL7MQc7vIsISBG8VQ8+IDQxpfQA==",
      "dev": true,
      "license": "BSD-3-Clause",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/strip-json-comments": {
      "version": "3.1.1",
      "resolved": "https://registry.npmjs.org/strip-json-comments/-/strip-json-comments-3.1.1.tgz",
      "integrity": "sha512-6fPc+R4ihwqP6N/aIv2f1gMH8lOVtWQHoqC4yK6oSDVVocumAsfCqjkXnqiYMhmMwS/mEHLp7Vehlt3ql6lEig==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=8"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    },
    "node_modules/supports-color": {
      "version": "7.2.0",
      "resolved": "https://registry.npmjs.org/supports-color/-/supports-color-7.2.0.tgz",
      "integrity": "sha512-qpCAvRl9stuOHveKsn7HncJRvv501qIacKzQlO/+Lwxc9+0q2wLyv4Dfvt80/DPn2pqOBsJdDiogXGR9+OvwRw==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "has-flag": "^4.0.0"
      },
      "engines": {
        "node": ">=8"
      }
    },
    "node_modules/sweetalert2": {
      "version": "11.23.0",
      "resolved": "https://registry.npmjs.org/sweetalert2/-/sweetalert2-11.23.0.tgz",
      "integrity": "sha512-cKzzbC3C1sIs7o9XAMw4E8F9kBtGXsBDUsd2JZ8JM/dqa+nzWwSGM+9LLYILZWzWHzX9W+HJNHyBlbHPVS/krw==",
      "license": "MIT",
      "funding": {
        "type": "individual",
        "url": "https://github.com/sponsors/limonte"
      }
    },
    "node_modules/tailwindcss": {
      "version": "4.1.13",
      "resolved": "https://registry.npmjs.org/tailwindcss/-/tailwindcss-4.1.13.tgz",
      "integrity": "sha512-i+zidfmTqtwquj4hMEwdjshYYgMbOrPzb9a0M3ZgNa0JMoZeFC6bxZvO8yr8ozS6ix2SDz0+mvryPeBs2TFE+w==",
      "dev": true,
      "license": "MIT"
    },
    "node_modules/tapable": {
      "version": "2.2.3",
      "resolved": "https://registry.npmjs.org/tapable/-/tapable-2.2.3.tgz",
      "integrity": "sha512-ZL6DDuAlRlLGghwcfmSn9sK3Hr6ArtyudlSAiCqQ6IfE+b+HHbydbYDIG15IfS5do+7XQQBdBiubF/cV2dnDzg==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=6"
      },
      "funding": {
        "type": "opencollective",
        "url": "https://opencollective.com/webpack"
      }
    },
    "node_modules/tar": {
      "version": "7.5.1",
      "resolved": "https://registry.npmjs.org/tar/-/tar-7.5.1.tgz",
      "integrity": "sha512-nlGpxf+hv0v7GkWBK2V9spgactGOp0qvfWRxUMjqHyzrt3SgwE48DIv/FhqPHJYLHpgW1opq3nERbz5Anq7n1g==",
      "dev": true,
      "license": "ISC",
      "dependencies": {
        "@isaacs/fs-minipass": "^4.0.0",
        "chownr": "^3.0.0",
        "minipass": "^7.1.2",
        "minizlib": "^3.1.0",
        "yallist": "^5.0.0"
      },
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/tar/node_modules/yallist": {
      "version": "5.0.0",
      "resolved": "https://registry.npmjs.org/yallist/-/yallist-5.0.0.tgz",
      "integrity": "sha512-YgvUTfwqyc7UXVMrB+SImsVYSmTS8X/tSrtdNZMImM+n7+QTriRXyXim0mBrTXNeqzVF0KWGgHPeiyViFFrNDw==",
      "dev": true,
      "license": "BlueOak-1.0.0",
      "engines": {
        "node": ">=18"
      }
    },
    "node_modules/tinyglobby": {
      "version": "0.2.15",
      "resolved": "https://registry.npmjs.org/tinyglobby/-/tinyglobby-0.2.15.tgz",
      "integrity": "sha512-j2Zq4NyQYG5XMST4cbs02Ak8iJUdxRM0XI5QyxXuZOzKOINmWurp3smXu3y5wDcJrptwpSjgXHzIQxR0omXljQ==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "fdir": "^6.5.0",
        "picomatch": "^4.0.3"
      },
      "engines": {
        "node": ">=12.0.0"
      },
      "funding": {
        "url": "https://github.com/sponsors/SuperchupuDev"
      }
    },
    "node_modules/type-check": {
      "version": "0.4.0",
      "resolved": "https://registry.npmjs.org/type-check/-/type-check-0.4.0.tgz",
      "integrity": "sha512-XleUoc9uwGXqjWwXaUTZAmzMcFZ5858QA2vvx1Ur5xIcixXIP+8LnFDgRplU30us6teqdlskFfu+ae4K79Ooew==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "prelude-ls": "^1.2.1"
      },
      "engines": {
        "node": ">= 0.8.0"
      }
    },
    "node_modules/update-browserslist-db": {
      "version": "1.1.3",
      "resolved": "https://registry.npmjs.org/update-browserslist-db/-/update-browserslist-db-1.1.3.tgz",
      "integrity": "sha512-UxhIZQ+QInVdunkDAaiazvvT/+fXL5Osr0JZlJulepYu6Jd7qJtDZjlur0emRlT71EN3ScPoE7gvsuIKKNavKw==",
      "dev": true,
      "funding": [
        {
          "type": "opencollective",
          "url": "https://opencollective.com/browserslist"
        },
        {
          "type": "tidelift",
          "url": "https://tidelift.com/funding/github/npm/browserslist"
        },
        {
          "type": "github",
          "url": "https://github.com/sponsors/ai"
        }
      ],
      "license": "MIT",
      "dependencies": {
        "escalade": "^3.2.0",
        "picocolors": "^1.1.1"
      },
      "bin": {
        "update-browserslist-db": "cli.js"
      },
      "peerDependencies": {
        "browserslist": ">= 4.21.0"
      }
    },
    "node_modules/uri-js": {
      "version": "4.4.1",
      "resolved": "https://registry.npmjs.org/uri-js/-/uri-js-4.4.1.tgz",
      "integrity": "sha512-7rKUyy33Q1yc98pQ1DAmLtwX109F7TIfWlW1Ydo8Wl1ii1SeHieeh0HHfPeL2fMXK6z0s8ecKs9frCuLJvndBg==",
      "dev": true,
      "license": "BSD-2-Clause",
      "dependencies": {
        "punycode": "^2.1.0"
      }
    },
    "node_modules/vite": {
      "version": "7.1.7",
      "resolved": "https://registry.npmjs.org/vite/-/vite-7.1.7.tgz",
      "integrity": "sha512-VbA8ScMvAISJNJVbRDTJdCwqQoAareR/wutevKanhR2/1EkoXVZVkkORaYm/tNVCjP/UDTKtcw3bAkwOUdedmA==",
      "dev": true,
      "license": "MIT",
      "dependencies": {
        "esbuild": "^0.25.0",
        "fdir": "^6.5.0",
        "picomatch": "^4.0.3",
        "postcss": "^8.5.6",
        "rollup": "^4.43.0",
        "tinyglobby": "^0.2.15"
      },
      "bin": {
        "vite": "bin/vite.js"
      },
      "engines": {
        "node": "^20.19.0 || >=22.12.0"
      },
      "funding": {
        "url": "https://github.com/vitejs/vite?sponsor=1"
      },
      "optionalDependencies": {
        "fsevents": "~2.3.3"
      },
      "peerDependencies": {
        "@types/node": "^20.19.0 || >=22.12.0",
        "jiti": ">=1.21.0",
        "less": "^4.0.0",
        "lightningcss": "^1.21.0",
        "sass": "^1.70.0",
        "sass-embedded": "^1.70.0",
        "stylus": ">=0.54.8",
        "sugarss": "^5.0.0",
        "terser": "^5.16.0",
        "tsx": "^4.8.1",
        "yaml": "^2.4.2"
      },
      "peerDependenciesMeta": {
        "@types/node": {
          "optional": true
        },
        "jiti": {
          "optional": true
        },
        "less": {
          "optional": true
        },
        "lightningcss": {
          "optional": true
        },
        "sass": {
          "optional": true
        },
        "sass-embedded": {
          "optional": true
        },
        "stylus": {
          "optional": true
        },
        "sugarss": {
          "optional": true
        },
        "terser": {
          "optional": true
        },
        "tsx": {
          "optional": true
        },
        "yaml": {
          "optional": true
        }
      }
    },
    "node_modules/which": {
      "version": "2.0.2",
      "resolved": "https://registry.npmjs.org/which/-/which-2.0.2.tgz",
      "integrity": "sha512-BLI3Tl1TW3Pvl70l3yq3Y64i+awpwXqsGBYWkkqMtnbXgrMD+yj7rhW0kuEDxzJaYXGjEW5ogapKNMEKNMjibA==",
      "dev": true,
      "license": "ISC",
      "dependencies": {
        "isexe": "^2.0.0"
      },
      "bin": {
        "node-which": "bin/node-which"
      },
      "engines": {
        "node": ">= 8"
      }
    },
    "node_modules/word-wrap": {
      "version": "1.2.5",
      "resolved": "https://registry.npmjs.org/word-wrap/-/word-wrap-1.2.5.tgz",
      "integrity": "sha512-BN22B5eaMMI9UMtjrGd5g5eCYPpCPDUy0FJXbYsaT5zYxjFOckS53SQDE3pWkVoWpHXVb3BrYcEN4Twa55B5cA==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=0.10.0"
      }
    },
    "node_modules/yallist": {
      "version": "3.1.1",
      "resolved": "https://registry.npmjs.org/yallist/-/yallist-3.1.1.tgz",
      "integrity": "sha512-a4UGQaWPH59mOXUYnAG2ewncQS4i4F43Tv3JoAM+s2VDAmS9NsK8GpDMLrCHPksFT7h3K6TOoUNn2pb7RoXx4g==",
      "dev": true,
      "license": "ISC"
    },
    "node_modules/yocto-queue": {
      "version": "0.1.0",
      "resolved": "https://registry.npmjs.org/yocto-queue/-/yocto-queue-0.1.0.tgz",
      "integrity": "sha512-rVksvsnNCdJ/ohGc6xgPwyN8eheCxsiLM8mxuE/t/mOVqJewPuO1miLpTHQiRgTKCLexL4MeAFVagts7HmNZ2Q==",
      "dev": true,
      "license": "MIT",
      "engines": {
        "node": ">=10"
      },
      "funding": {
        "url": "https://github.com/sponsors/sindresorhus"
      }
    }
  }
}

``


---

## README.md
``markdown
# ?? YTTA-AJA: Anonymous Messaging App (React + Google Apps Script)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Technology: React](https://img.shields.io/badge/Frontend-React%20%7C%20Vite-61DAFB?logo=react)](https://react.dev/)
[![Technology: Tailwind CSS](https://img.shields.io/badge/Styling-Tailwind%20CSS-06B6D4?logo=tailwindcss)](https://tailwindcss.com/)
[![Technology: Google Apps Script](https://img.shields.io/badge/Backend-Google%20Apps%20Script-3F51B5?logo=google)](https://developers.google.com/apps-script)

A simple web application for sending and receiving secret messages **anonymously**. This project serves as an experiment in utilizing **Google Sheets as a Serverless Database** through a **Google Apps Script** (GAS) web API endpoint.

## ?? Key Features

* **Anonymous Messaging**: Senders don't need to log in or register.
* **Unique Share Link**: Each registered user gets a unique link (`/send/:userId`) to receive messages.
* **Authentication**: Simple Login/Register using a `userId` and a secure `loginKey`.
* **Modern UI**: Elegant **Cyber/Neon Theme** (Red & Orange Palette) powered by **Tailwind CSS**.
* **Modal Viewer**: Full message viewer with a cool *backdrop blur* effect.

## ??? Tech Stack

### Frontend
* **React.js (Vite)**
* **React Router DOM**
* **Tailwind CSS** (Custom Neon Red/Orange Palette)
* **SweetAlert2** (For stylish notifications)
* **React Bootstrap Icons**

### Backend / Database
* **Google Apps Script (GAS)**: Acts as the *Web API* to handle `register`, `login`, and `send` requests.
* **Google Sheets**: Used as the primary *database* to store user and message data.

## ?? Installation and Setup

### 1. Backend (Google Apps Script)

1.  **Prepare Google Sheets:** Create a new Google Sheet with two tabs: `Users` and `Messages`.
    * **`Users` Sheet** (Required columns): `userId`, `loginKey`, `namaTampilan`
    * **`Messages` Sheet** (Required columns): `recipientId`, `Pengirim`, `Pesan`, `Tanggal`
2.  **Deploy Apps Script:** Copy the Google Apps Script code (your **`code.gs`**) into the GAS editor.
3.  **Deploy as Web App:** Publish the script and note down the resulting **`ANONYMOUS_API_URL`**.

### 2. Frontend (React)

1.  **Clone the Repository:**
    ```bash
    git clone https://github.com/Arman1862/ytta-aja.git
    cd ytta-aja
    ```
2.  **Install Dependencies:**
    ```bash
    npm install
    ```
3.  **Configure API URL:**
    Create the file **`src/config/api.js`** (if it doesn't exist) and input your deployed Apps Script URL:

    ```javascript
    // src/config/api.js
    export const ANONYMOUS_API_URL = "[https://script.google.com/macros/s/](https://script.google.com/macros/s/)[YOUR_GAS_ID]/exec"; 
    // Replace [YOUR_GAS_ID] with your deployment ID
    ```
4.  **Run the Application:**
    ```bash
    npm run dev
    ```
    The app will be running at `http://localhost:5173/` (or another port).

## ?? Design & Theme

This application uses a **Neon Cyberpunk** theme with a dominant color palette of Neon Red (`#FF3366`) and Bright Orange (`#FF9933`). All styling is managed via **Tailwind CSS**.

* Custom colors can be found in `tailwind.config.cjs`.
* A *glassmorphism* effect is applied to most main cards (`bg-white/5 backdrop-blur-xl`).

## ????? Contributor

This project was developed by **Arman - Muhammad Arjuna Mahendratama** while studying at SMKN 53 Jakarta.

> *Feedback and suggestions are highly appreciated!*
``


---

## src/App.css
``css

@keyframes blob {
    0%, 100% {
      transform: translate(0, 0) scale(1);
    }
    33% {
      transform: translate(30px, -50px) scale(1.1);
    }
    66% {
      transform: translate(-20px, 20px) scale(0.9);
    }
  }
  
  .drop-shadow-neon {
    filter: drop-shadow(0 0 5px rgba(232, 62, 140, 0.7)) 
            drop-shadow(0 0 10px rgba(99, 102, 241, 0.5));
  }
  
  .animation-delay-2000 {
    animation-delay: 2s;
  }
  
  .animate-blob {
    animation: blob 7s infinite alternate;
  }
``


---

## src/App.jsx
``jsx
import { Routes, Route, useParams } from 'react-router-dom';
import Home from './components/Home';
import LoginForm from './components/LoginForm';
import RegisterForm from './components/RegisterForm';
import KirimPesanAnonim from './components/KirimPesanAnonim';
import Dashboard from './components/Dashboard';
import Footer from './components/Footer';
import './App.css'; 

const SendMessagePage = () => {
  const { recipientId } = useParams();

  return (
      <KirimPesanAnonim recipientId={recipientId} />
  );
};

function App() {
  return (
    <div className="flex flex-col min-h-screen bg-gray-950">
      <main className="flex-grow">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/login" element={<LoginForm />} />
          <Route path="/register" element={<RegisterForm />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/send/:recipientId" element={<SendMessagePage />} />
        </Routes>
      </main>
      <Footer />
    </div>
  );
}

export default App;
``


---

## src/components/Dashboard.jsx
``jsx
import React, { useState, useEffect } from "react";
import { useNavigate, Link } from "react-router-dom";
import Swal from "sweetalert2";
import TampilPesanAnonim from "./TampilPesanAnonim";
import { Mailbox2, Clipboard, X } from "react-bootstrap-icons";

const MessageModal = ({ pesan, nomorPesan, onClose }) => {
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    if (pesan) {
      const timeout = setTimeout(() => setIsVisible(true), 10);
      return () => clearTimeout(timeout);
    } else {
      setIsVisible(false);
    }
  }, [pesan]);
  
  if (!pesan && !isVisible) return null;

  const handleClose = () => {
      setIsVisible(false);
      setTimeout(onClose, 300);
  };

  const formattedDate = pesan ? new Date(pesan.Tanggal).toLocaleString() : "";

  return (
    <div 
      className={`fixed inset-0 z-50 flex items-center justify-center p-4 transition-opacity duration-300 
                 ${isVisible ? "opacity-100 backdrop-blur-sm bg-black/70" : "opacity-0 pointer-events-none bg-black/0"}`}
      onClick={handleClose}
    >
      <div 
        className={`bg-white/10 backdrop-blur-xl border border-red-500/30 rounded-3xl shadow-lg shadow-red-500/20 p-6 w-full max-w-sm mx-auto transition-all duration-300 
                    ${isVisible ? "scale-100 opacity-100" : "scale-90 opacity-0"}`}
        onClick={(e) => e.stopPropagation()}
      >
        <div className="flex justify-between items-start mb-4">
          <h3 className="text-xl font-bold text-red-neon">
            Pesan Rahasia #{nomorPesan || "#"} 
          </h3>
          <button onClick={handleClose} className="text-gray-400 hover:text-red-400 transition">
            <X className="text-2xl" />
          </button>
        </div>
        
        <p className="text-white text-base mb-4 whitespace-pre-wrap">{pesan.Pesan}</p>

        <div className="text-xs border-t border-red-500/30 pt-3 text-gray-400">
            <p className="font-semibold text-orange-300">Pengirim: {pesan.Pengirim}</p>
            <p>Tanggal: {formattedDate}</p>
        </div>
      </div>
    </div>
  );
};


export default function Dashboard() {
  const [userAuth, setUserAuth] = useState(null);
  const [refreshTrigger, setRefreshTrigger] = useState(0); 
  const [selectedMessageData, setSelectedMessageData] = useState(null); 
  const navigate = useNavigate();

  useEffect(() => {
    const storedData = localStorage.getItem("userAuth");
    if (storedData) {
      setUserAuth(JSON.parse(storedData));
    } else {
      navigate("/login");
    }
  }, [navigate]);

  const handleCopyLink = () => {
    if (userAuth && userAuth.userId) {
      const shareLink = `${window.location.origin}/send/${userAuth.userId}`; 
      navigator.clipboard.writeText(shareLink).then(() => {
        Swal.fire({
          title: "Link Berhasil Disalin!",
          text: `Bagikan link: ${shareLink}`,
          icon: "success",
          timer: 2500,
          showConfirmButton: false,
          customClass: {
            popup: "text-white bg-slate-800 rounded-xl",
          }
        });
      }).catch(err => {
        console.error("Failed to copy:", err);
        Swal.fire("Oops!", "Gagal menyalin link.", "error");
      });
    }
  };

  const handleSelectPesan = (pesan, nomor) => {
    setSelectedMessageData({ pesan: pesan, nomorPesan: nomor });
  };
  
  const handleCloseModal = () => {
    setSelectedMessageData(null); 
  };

  if (!userAuth) {
    return <div className="min-h-screen bg-gray-950 text-white flex items-center justify-center"><p>Loading...</p></div>;
  }

  const shareLink = `${window.location.origin}/send/${userAuth.userId}`;

    
  const handleLogout = () => {
    localStorage.removeItem("userAuth");
  };

  return (
    <div className="min-h-screen bg-gray-950 text-white flex items-center justify-center p-4 relative overflow-hidden">
      
      <div className="absolute top-0 left-0 w-80 h-80 bg-red-500/20 rounded-full mix-blend-multiply filter blur-3xl opacity-50 animate-blob"></div>
      <div className="absolute bottom-0 right-0 w-80 h-80 bg-orange-500/20 rounded-full mix-blend-multiply filter blur-3xl opacity-50 animate-blob animation-delay-2000"></div>

      <div className="relative z-10 bg-white/5 backdrop-blur-xl border border-red-500/30 rounded-3xl shadow-lg shadow-red-500/10 p-8 w-full max-w-md mx-auto my-8 transition-all duration-500 hover:shadow-red-500/20">
        
        <Mailbox2 className="text-red-neon text-5xl mx-auto mb-4" />
        <h1 className="text-3xl md:text-3xl text-orange-400 font-bold text-center mb-6">
          Welcome, <span className="text-orange-400">{userAuth.namaTampilan}!</span>
        </h1>
        
        <div className="bg-white/10 backdrop-blur-md border border-orange-500/20 rounded-xl shadow-inner shadow-red-500/5 p-4 mb-8">
          <h2 className="text-xl font-semibold mb-3 text-red-400">Bagikan Link Rahasiamu</h2>
          <div className="flex flex-col sm:flex-row items-stretch space-y-3 sm:space-y-0 sm:space-x-3">
            <input
              type="text"
              readOnly
              value={shareLink}
              className="w-full px-3 py-2 border rounded-lg bg-white/5 border-orange-500/20 text-white placeholder-gray-400 text-sm flex-grow focus:ring-red-500 focus:border-red-500 transition-all duration-300"
            />
            <button
              onClick={handleCopyLink}
              className="w-full sm:w-auto px-4 py-2 font-semibold rounded-lg 
                         bg-gradient-to-r from-red-600 to-orange-600 
                         text-white 
                         shadow-md shadow-red-500/30 
                         hover:from-red-500 hover:to-orange-500 
                         transition duration-300 flex items-center justify-center text-sm"
            >
              <Clipboard className="mr-2 text-lg" />
              Copy Link
            </button>
          </div>
        </div>

        <div className="text-left mb-6">
             <h2 className="text-xl font-semibold mb-3 text-red-400">Kotak Masuk Anonim</h2>
             <TampilPesanAnonim 
                 refreshTrigger={refreshTrigger} 
                 onSelectPesan={handleSelectPesan} 
             />
        </div>
       
        <div className="text-center mt-6">
            <Link 
                to="/login" 
                onClick={handleLogout}
                className="text-sm text-gray-400 hover:text-red-400 font-semibold transition-colors"
            >
                Logout
            </Link>
        </div>
      </div>
      
      {selectedMessageData && (
          <MessageModal 
              pesan={selectedMessageData.pesan} 
              nomorPesan={selectedMessageData.nomorPesan}
              onClose={handleCloseModal}
          />
      )}

    </div>
  );
}
``


---

## src/components/Footer.jsx
``jsx
import React from "react";

export default function Footer() {
  return (
    <footer className="w-full text-center p-4 text-orange-400 text-sm bg-gray-950 backdrop-blur-md ">
      <p>
        Created by: 
        <a 
          href="https://www.instagram.com/cyvix4102/" 
          target="_blank" 
          rel="noopener noreferrer" 
          className="font-bold from-red-400 to-orange-400 hover:text-orange-cyber transition duration-200 mx-1"
        >
          @cyvix4102
        </a> 
        <span className="text-orange-400">|</span> 
        <a 
          href="https://www.instagram.com/calx4102" 
          target="_blank" 
          rel="noopener noreferrer" 
          className="font-bold text-red-neon hover:text-orange-cyber transition duration-200 ml-1"
        >
          @calx4102
        </a>
      </p>
    </footer>
  );
}
``


---

## src/components/Home.jsx
``jsx
import React from "react";
import { Link } from "react-router-dom";
import { PersonCircle } from "react-bootstrap-icons"; 

export default function Home() {
  return (
    <div className="min-h-screen bg-gray-950 text-white flex flex-col items-center justify-center p-4 relative overflow-hidden">
      
      <div className="absolute top-0 left-0 w-80 h-80 bg-red-500/20 rounded-full mix-blend-multiply filter blur-3xl opacity-50 animate-blob"></div>
      <div className="absolute bottom-0 right-0 w-80 h-80 bg-orange-500/20 rounded-full mix-blend-multiply filter blur-3xl opacity-50 animate-blob animation-delay-2000"></div>

      <div className="relative z-10 bg-white/5 backdrop-blur-xl border border-red-500/30 rounded-3xl shadow-lg shadow-red-500/10 p-8 w-full max-w-sm mx-auto my-8 text-center transition-all duration-500 hover:shadow-red-500/20">
        
        <PersonCircle className="text-red-400 text-6xl mx-auto mb-4" /> 
        <h1 className="text-4xl font-extrabold mb-2 text-transparent bg-clip-text bg-gradient-to-r from-red-400 to-orange-400 uppercase tracking-wider">
          YTTA Aja
        </h1>
        <p className="text-md mb-8 text-gray-400">
          Kirim dan terima pesan rahasia tanpa nama.
        </p>

        <div className="flex flex-col space-y-4">
          
          <Link 
            to="/login" 
            className="w-full px-6 py-3 font-bold rounded-xl 
                       bg-gradient-to-r from-red-600 to-orange-600 
                       text-white 
                       shadow-md shadow-red-500/30 
                       hover:from-red-500 hover:to-orange-500 
                       hover:shadow-lg hover:shadow-red-500/50
                       transition duration-300 transform hover:scale-[1.02] 
                       text-center uppercase tracking-wider"
          >
            Login
          </Link>
          
          <Link 
            to="/register" 
            className="w-full px-6 py-3 font-semibold rounded-xl 
                       bg-transparent 
                       border-2 border-red-500 
                       text-red-400 
                       hover:bg-red-500/10 
                       hover:text-white
                       transition duration-300 transform hover:scale-[1.02]
                       text-center uppercase tracking-wider"
          >
            Register
          </Link>
        </div>
      </div>
    </div>
  );
}
``


---

## src/components/KirimPesanAnonim.jsx
``jsx
import { useState } from "react";
import Swal from "sweetalert2";
import { ANONYMOUS_API_URL } from "../config/api"; 
import { Envelope } from "react-bootstrap-icons";

export default function KirimPesanAnonim({ onPesanTerkirim, recipientId }) {
  const [pesan, setPesan] = useState("");
  const [pengirim, setPengirim] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const [isSent, setIsSent] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);

    const formData = new FormData();
    formData.append("action", "send"); 
    formData.append("pesan", pesan);
    formData.append("pengirim", pengirim.trim() === "" ? "Anonim" : pengirim.trim()); 
    if (recipientId) {
      formData.append("recipientId", recipientId);
    }

    try {
      const response = await fetch(ANONYMOUS_API_URL, { method: "POST", body: formData });
      const result = await response.json();

      if (response.ok && result.result === "success") {
        setIsSent(true); 
        if (onPesanTerkirim) onPesanTerkirim();
      } else {
        const errorText = result.message || "Pesan gagal terkirim. Cek log Apps Script untuk detailnya."; 
        Swal.fire("Gagal!", errorText, "error");
      }
    } catch (error) {
      console.error("Fetch error:", error);
      Swal.fire("Error!", "Koneksi gagal. Cek jaringan atau URL API.", "error");
    } finally {
      setIsLoading(false);
    }
  };

  const handleSendAgain = () => {
    setIsSent(false);
    setPesan("");
    setPengirim("");
  };

  return (
    <div className="min-h-screen bg-gray-950 text-white flex items-center justify-center p-4 relative overflow-hidden">
      
      <div className="absolute top-0 left-0 w-80 h-80 bg-red-500/20 rounded-full mix-blend-multiply filter blur-3xl opacity-50 animate-blob"></div>
      <div className="absolute bottom-0 right-0 w-80 h-80 bg-orange-500/20 rounded-full mix-blend-multiply filter blur-3xl opacity-50 animate-blob animation-delay-2000"></div>

      <div className="relative z-10 bg-white/5 backdrop-blur-xl border border-red-500/30 rounded-3xl shadow-lg shadow-red-500/10 p-8 w-full max-w-sm mx-auto my-8 transition-all duration-500 hover:shadow-red-500/20">
        
        {isSent ? (
          <div className="text-center py-8">
            <h3 className="text-4xl font-extrabold mb-3 text-transparent bg-clip-text bg-gradient-to-r from-orange-400 to-red-400">
              Pesan Terkirim!
            </h3>
            <p className="text-lg text-gray-300 mb-8">
              Terima kasih sudah mengirim pesan anonim kepada @{recipientId}!
            </p>
            <button
              onClick={handleSendAgain}
              className="w-full px-6 py-3 font-bold rounded-xl 
                         bg-gradient-to-r from-red-600 to-orange-600 
                         text-white 
                         shadow-md shadow-red-500/30 
                         hover:from-red-500 hover:to-orange-500 
                         hover:shadow-lg hover:shadow-red-500/50
                         transition duration-300 transform hover:scale-[1.02] 
                         uppercase tracking-wider"
            >
              Kirim Pesan Lagi
            </button>
            <a href="/" className="block mt-4 text-sm text-red-400 hover:text-orange-400 transition-colors">
              Kembali ke Beranda
            </a>
          </div>
        ) : (
          <>
            <Envelope className="text-red-400 text-5xl mx-auto mb-4" />
            <h2 className="text-3xl font-bold text-center mb-6 text-transparent bg-clip-text bg-gradient-to-r from-red-400 to-orange-400">
              Kirim Pesan Anonim
            </h2>
            <p className="text-center text-gray-300 mb-6">
              Tulis pesan rahasia untuk @{recipientId}
            </p>
            <form onSubmit={handleSubmit} className="space-y-6">
              <div>
                <label htmlFor="pengirim" className="block text-sm font-medium mb-2 text-gray-300">Nama Pengirim (Opsional)</label>
                <input
                  type="text"
                  id="pengirim"
                  className="w-full px-4 py-3 border rounded-xl bg-white/10 border-red-500/20 text-white placeholder-gray-400 focus:ring-orange-500 focus:border-orange-500 transition-all duration-300"
                  placeholder="Contoh: Secret Admirer / Kosongin"
                  value={pengirim}
                  onChange={(e) => setPengirim(e.target.value)}
                  disabled={isLoading}
                  name="pengirim"
                />
              </div>
              <div>
                <label htmlFor="pesan" className="block text-sm font-medium mb-2 text-gray-300">Pesan Anonim</label>
                <textarea
                  id="pesan"
                  className="w-full px-4 py-3 border rounded-xl bg-white/10 border-orange-500/20 text-white placeholder-gray-400 focus:ring-red-500 focus:border-red-500 transition-all duration-300"
                  rows="4"
                  placeholder="Tulis pesan rahasia kamu di sini..."
                  value={pesan}
                  onChange={(e) => setPesan(e.target.value)}
                  required
                  disabled={isLoading}
                  name="pesan"
                ></textarea>
              </div>
              <button
                type="submit"
                className="w-full px-6 py-3 font-bold rounded-xl 
                           bg-gradient-to-r from-red-600 to-orange-600 
                           text-white 
                           shadow-md shadow-red-500/30 
                           hover:from-red-500 hover:to-orange-500 
                           hover:shadow-lg hover:shadow-red-500/50
                           transition duration-300 transform hover:scale-[1.02] 
                           uppercase tracking-wider"
                disabled={isLoading}
              >
                {isLoading ? "Mengirim..." : "Kirim Pesan"}
              </button>
            </form>
          </>
        )}
      </div>
    </div>
  );
}
``


---

## src/components/LoginForm.jsx
``jsx
import React, { useState } from "react";
import { Link, useNavigate } from "react-router-dom";
import Swal from "sweetalert2";
import { Lock } from "react-bootstrap-icons";
import { ANONYMOUS_API_URL } from "../config/api";

export default function LoginForm() {
  const [userId, setUserId] = useState("");
  const [loginKey, setLoginKey] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const navigate = useNavigate();
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);

    const loginUrl = `${ANONYMOUS_API_URL}?action=login&userId=${encodeURIComponent(
      userId.trim()
    )}&loginKey=${encodeURIComponent(loginKey.trim())}`;

    try {
      const response = await fetch(loginUrl, { method: "GET" });
      const result = await response.json();

      if (response.ok && result.result === "success") {
        const userAuthData = {
          userId: result.profile.userId,
          loginKey: loginKey.trim(),
          namaTampilan: result.profile.namaTampilan,
        };

        localStorage.setItem("userAuth", JSON.stringify(userAuthData));

        Swal.fire({
          title: "Login Sukses!",
          text: `Selamat datang, ${result.profile.namaTampilan}!`,
          icon: "success",
          timer: 1500,
          showConfirmButton: false,
          background: "#0A0A0A",
          color: "#ffffff"
        }).then(() => {
          navigate("/dashboard");
        });
      } else {
        const errorText =
          result.message || "Login gagal. Cek User ID dan Kunci Rahasiamu.";
        Swal.fire({
          title: "Gagal!",
          text: errorText,
          icon: "error",
          background: "#0A0A0A",
          color: "#ffffff",
          confirmButtonColor: "#FF3366"
        });
      }
    } catch (error) {
      console.error("Fetch error:", error);
      Swal.fire({
        title: "Error!",
        text: "Koneksi gagal. Cek jaringan atau URL API.",
        icon: "error",
        background: "#0A0A0A",
        color: "#ffffff",
        confirmButtonColor: "#FF3366"
      });
    } finally {
      setIsLoading(false);
    }
  };
  return (
    <div className="min-h-screen bg-gray-950 text-white flex items-center justify-center p-4 relative overflow-hidden">
      

      <div className="relative z-10 bg-white/5 backdrop-blur-xl border border-red-500/30 rounded-3xl shadow-lg shadow-red-500/10 p-8 w-full max-w-sm mx-auto my-8 transition-all duration-500 hover:shadow-red-500/20">
        
        <Lock className="text-orange-400 text-5xl mx-auto mb-4" />
        <h1 className="text-3xl text-center font-extrabold mb-8 text-transparent bg-clip-text bg-gradient-to-r from-red-400 to-orange-400 uppercase tracking-wider">
          Login
        </h1>

        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label htmlFor="userId" className="block text-sm font-medium mb-2 text-gray-300 text-left">User ID</label>
            <input
              type="text"
              id="userId"
              className="w-full px-4 py-3 border rounded-xl bg-white/10 border-red-500/20 text-white placeholder-gray-400 focus:ring-orange-500 focus:border-orange-500 transition-all duration-300"
              placeholder="Masukkan User ID kamu"
              value={userId}
              onChange={(e) => setUserId(e.target.value)}
              required
              disabled={isLoading}
            />
          </div>
          
          <div className="mb-6">
            <label htmlFor="loginKey" className="block text-sm font-medium mb-2 text-gray-300 text-left">Kunci Rahasia</label>
            <input
              type="password"
              id="loginKey"
              className="w-full px-4 py-3 border rounded-xl bg-white/10 border-orange-500/20 text-white placeholder-gray-400 focus:ring-red-500 focus:border-red-500 transition-all duration-300"
              placeholder="Masukkan kunci rahasia"
              value={loginKey}
              onChange={(e) => setLoginKey(e.target.value)}
              required
              disabled={isLoading}
            />
          </div>
          
          <button
            type="submit"
            className="w-full px-6 py-3 font-bold rounded-xl 
                       bg-gradient-to-r from-red-600 to-orange-600 
                       text-white 
                       shadow-md shadow-red-500/30 
                       hover:from-red-500 hover:to-orange-500 
                       hover:shadow-lg hover:shadow-red-500/50
                       transition duration-300 transform hover:scale-[1.02] 
                       uppercase tracking-wider"
            disabled={isLoading}
          >
            {isLoading ? "Logging in..." : "Login"}
          </button>
        </form>
        
        <p className="text-center mt-6 text-sm text-gray-400">
          Belum punya akun?{" "}
          <Link to="/register" className="text-red-400 hover:text-red-300 font-semibold">Register di sini</Link>
        </p>
      </div>
    </div>
  );
}
``


---

## src/components/RegisterForm.jsx
``jsx
import React, { useState } from "react";
import { Link, useNavigate } from "react-router-dom";
import Swal from "sweetalert2";
import { PersonAdd } from "react-bootstrap-icons";
import { ANONYMOUS_API_URL } from "../config/api";

export default function RegisterForm() {
  const [userId, setUserId] = useState("");
  const [displayName, setDisplayName] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const navigate = useNavigate();
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);

    const formData = new FormData();
    formData.append("action", "register");
    formData.append("userId", userId.trim()); 
    formData.append("namaTampilan", displayName.trim()); 

    try {
      const response = await fetch(ANONYMOUS_API_URL, { 
        method: "POST", 
        body: formData 
      });

      const result = await response.json(); 

      if (response.ok && result.result === "success" && result.data.loginKey) {
        const loginKey = result.data.loginKey;

        Swal.fire({
          title: "Registrasi Berhasil!",
          icon: "success",
          html: `
            <div class="text-left text-white my-4">
                <p>Akunmu (@${result.data.userId}) berhasil dibuat.</p>
                <p class="font-bold mt-3 mb-2">SIMPAN KUNCI RAHASIA INI:</p>
                <div class="flex items-center bg-gray-900 p-3 rounded-lg">
                    <code class="flex-grow text-orange-cyber font-mono">${loginKey}</code>
                    <button id="copy-key-btn" class="ml-4 px-3 py-1 rounded-lg bg-red-neon hover:opacity-80 text-white font-semibold">
                    Copy
                    </button>
                </div>
            </div>
          `,
          confirmButtonText: "Mengerti, Lanjut Login",
          background: "#0A0A0A",
          color: "#ffffff",
          confirmButtonColor: "#FF3366",
          didOpen: () => {
            const copyBtn = document.getElementById("copy-key-btn");
            if (copyBtn) {
              copyBtn.addEventListener("click", () => {
                navigator.clipboard.writeText(loginKey).then(() => {
                  copyBtn.textContent = "Copied!";
                  copyBtn.disabled = true;
                }).catch(err => {
                  console.error("Failed to copy key:", err);
                  copyBtn.textContent = "Failed!";
                });
              });
            }
          }
        }).then(() => {
          navigate("/login");
        });
      } else {
        const errorText = result.message || "Gagal! Cek log Apps Script untuk detailnya."; 
        Swal.fire({
            title: "Gagal!", 
            text: errorText, 
            icon: "error",
            background: "#0A0A0A",
            color: "#ffffff",
            confirmButtonColor: "#FF3366"
        });
      }
    } catch (error) {
      console.error("Registration error:", error);
      Swal.fire({
          title: "Error", 
          text: "An error occurred during registration.", 
          icon: "error",
          background: "#0A0A0A",
          color: "#ffffff",
          confirmButtonColor: "#FF3366"
      });
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-950 text-white flex items-center justify-center p-4 relative overflow-hidden">
      

      <div className="relative z-10 bg-white/5 backdrop-blur-xl border border-red-500/30 rounded-3xl shadow-lg shadow-red-500/10 p-8 w-full max-w-sm mx-auto my-8 transition-all duration-500 hover:shadow-red-500/20">
        
        <PersonAdd className="text-red-400 text-5xl mx-auto mb-4" />
        <h1 className="text-3xl text-center font-extrabold mb-8 text-transparent bg-clip-text bg-gradient-to-r from-red-400 to-orange-400 uppercase tracking-wider">
          Register
        </h1>

        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label htmlFor="userId" className="block text-sm font-medium mb-2 text-gray-300 text-left">User ID</label>
            <input
              type="text"
              id="userId"
              className="w-full px-4 py-3 border rounded-xl bg-white/10 border-red-500/20 text-white placeholder-gray-400 focus:ring-orange-500 focus:border-orange-500 transition-all duration-300"
              placeholder="User ID unik (tanpa spasi)"
              value={userId}
              onChange={(e) => setUserId(e.target.value)}
              required
              disabled={isLoading}
            />
          </div>
          
          <div className="mb-6">
            <label htmlFor="displayName" className="block text-sm font-medium mb-2 text-gray-300 text-left">Nama Tampilan</label>
            <input
              type="text"
              id="displayName"
              className="w-full px-4 py-3 border rounded-xl bg-white/10 border-orange-500/20 text-white placeholder-gray-400 focus:ring-red-500 focus:border-red-500 transition-all duration-300"
              placeholder="Nama kamu di profil"
              value={displayName}
              onChange={(e) => setDisplayName(e.target.value)}
              required
              disabled={isLoading}
            />
          </div>
          
          <button
            type="submit"
            className="w-full px-6 py-3 font-bold rounded-xl 
                       bg-gradient-to-r from-red-600 to-orange-600 
                       text-white 
                       shadow-md shadow-red-500/30 
                       hover:from-red-500 hover:to-orange-500 
                       hover:shadow-lg hover:shadow-red-500/50
                       transition duration-300 transform hover:scale-[1.02] 
                       uppercase tracking-wider"
            disabled={isLoading}
          >
            {isLoading ? "Registering..." : "Register"}
          </button>
        </form>
        
        <p className="text-center mt-6 text-sm text-gray-400">
          Sudah punya akun?{" "}
          <Link to="/login" className="text-red-400 hover:text-red-300 font-semibold">Login di sini</Link>
        </p>
      </div>
    </div>
  );
}
``


---

## src/components/TampilPesanAnonim.jsx
``jsx
import { useState, useEffect } from "react";
import { Person } from "react-bootstrap-icons";
import { ANONYMOUS_API_URL } from "../config/api";

export default function TampilPesanAnonim({ refreshTrigger, onSelectPesan }) { 
  const [pesanAnonim, setPesanAnonim] = useState([]);
  const [isLoading, setIsLoading] = useState(true);

  const fetchPesanAnonim = async () => {
    setIsLoading(true);
    
    const userAuth = JSON.parse(localStorage.getItem("userAuth"));
    
    if (!userAuth || !userAuth.userId || !userAuth.loginKey) {
        console.error("User not authenticated.");
        setIsLoading(false);
        return; 
    }

    const fetchUrl = `${ANONYMOUS_API_URL}?action=login&userId=${encodeURIComponent(userAuth.userId)}&loginKey=${encodeURIComponent(userAuth.loginKey)}`;

    try {
      const response = await fetch(fetchUrl, { method: "GET" });
      
      if (!response.ok) throw new Error("Gagal mengambil data dari server.");
      
      const data = await response.json();
      
      if (data.result === "success" && data.messages) {
          setPesanAnonim(data.messages.reverse()); 
      } else {
          console.error("Fetch gagal:", data.message || "Respons tidak valid.");
          setPesanAnonim([]); 
      }

    } catch (error) {
      console.error("Gagal mengambil pesan:", error);
    } finally {
      setIsLoading(false);
    }
  };


  useEffect(() => {
    fetchPesanAnonim();
  }, [refreshTrigger]);

  return (
    <div className="w-full">
      <div className="max-h-[50vh] overflow-y-auto pr-2">
        {isLoading ? (
          <div className="text-center text-white/70 py-4">
            <p>Loading messages...</p>
          </div>
        ) : (
          pesanAnonim.length > 0 ? (
            [...pesanAnonim].reverse().map((pesan, i) => { 
              const totalPesan = pesanAnonim.length;
              const nomorPesan = totalPesan - i; 

              return ( 
                <div 
                  key={i} 
                  className="p-3 my-3 rounded-xl bg-white/10 border border-orange-500/10 shadow-lg shadow-red-500/5 flex items-start space-x-3 transition-all duration-300 hover:bg-white/15 cursor-pointer"
                  onClick={() => onSelectPesan(pesan, nomorPesan  )} 
                >
                  <Person className="text-orange-400 text-xl flex-shrink-0 mt-1" />
                  <div className="flex-grow text-left">
                    <p className="mb-1 text-base text-red-400 font-bold">Pesan Rahasia #{nomorPesan}</p>
                    <small className="text-white/50 text-xs">
                      dari **{pesan.Pengirim}** • {new Date(pesan.Tanggal).toLocaleString()}
                    </small>
                  </div>
                </div>
              ); 
            })
          ) : (
            <div className="text-center text-gray-500 py-10">
              <p>Belum ada pesan anonim. Bagikan linkmu!</p>
            </div>
          )
        )}
      </div>
    </div>
  );
}
``


---

## src/config/api.js
``javascript
// export const ANONYMOUS_API_URL = "";
export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbwx76IZrTLK5AeOciHgTUlEBJmTqC3no3E_PyCGkvjbLM-j1LaK3DamC2n3FJ0-cA3D/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbwuLDqLvtc09n9xEyFuK7TzM9fAD85Vz-Sg_pmtjgtM5g8Zhmy82PFBs8yv6pi3beLE/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbyyi7pGfG1zA4kBC2KcRKqJmMdMhWBFOjTE4EPxEFfS4LL2Ue-OygOQbBLs-c4Xtv-z/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbwbEa3Lx2G2MLu4kBL7E7Vw7rZWlNfQpBA8lPip6z_NANswfPFOVMJQU3umQsVSMl-8/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbzGEXdAo3-qM_CCmDBldujMNVSrbRYofP983o53YYdKsGB4kAg5Ufa05_AjMEJx6CEX/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbw01JgRQAtvgfmuRIJEtWI9vxG8qC6T3sVPYtS0yt1UjrE5Z7jA4b0kK2NiFCA3DJqr/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbzQq7AE6mPKL5wcfvF6Euqj95sZzIdQ9dkyDrkIEwXG6uv-uBbmeIo9LMuW_PnuGWCk/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbz6XO2fpJfgOfsSdgd7PzhTJEdHJUrzj1vZKnFTMAju75oYCIo07mzV18pzr3yfJps4/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbw3rDLb5j7Iw1KlDTt3o6gCdEROQk6nMhVaoyJyqj5J8TXycB_SeB_5qPL3BXyoFH_m/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbxbVKUSMfG0i8E89b1uersHGhh0PlNV9Xndr8A65E7PgrNEczzzii4O-ktNSEaJVDm3/exec";
// export const ANONYMOUS_API_URL = "https://script.google.com/macros/s/AKfycbzNDrEwKXBRRXIPu0xzQl0xxpCuRdMLP4iRBkFasO9SUk30qeuLZF_WoK5Z54RPtQmh/exec";
``


---

## src/index.css
``css
@import "tailwindcss";
``


---

## src/main.jsx
``jsx
import { StrictMode } from "react"
import { createRoot } from "react-dom/client"
import { BrowserRouter } from "react-router-dom";
import "./index.css"
import App from "./App.jsx"

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>,
)
``


---

## tailwind.config.cjs
``javascript
/** @type {import("tailwindcss").Config} */
module.exports = {
  content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
        "deep-indigo": "#0A0A0A",
        "red-neon": "#FF3366",
        "orange-cyber": "#FF9933",
        "glass-border-main": "#F87171",

        "deep-orange": "#1A0E0B",
        "orange-primary": "#DD6B20",
        "orange-neon": "#F97316",
        "amber-glow": "#FBBF24",
        "glass-border-orange": "#F59E0B",
      },
      boxShadow: {
        "neon-red": "0 0 10px #FF3366, 0 0 20px #FF3366",
        "neon-orange": "0 0 10px #FF9933, 0 0 20px #FF9933",
      },
      keyframes: {
        blob: {
          "0%, 100%": { transform: "translate(0px, 0px) scale(1)" },
          "33%": { transform: "translate(30px, -50px) scale(1.1)" },
          "66%": { transform: "translate(-20px, 20px) scale(0.9)" },
        },
      },
      animation: {
        blob: "blob 7s infinite",
      },
    },
  },
  plugins: [],
};
``


---

## tailwind.config.js
``javascript
/** @type {import("tailwindcss").Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
``


---

## vercel.json
``json
{
    "rewrites": [
      {
        "source": "/(.*)",
        "destination": "/index.html"
      }
    ]
  }
``


---

## vite.config.js
``javascript
import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"
import tailwindcss from "@tailwindcss/vite"

// https://vite.dev/config/
export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
  ],
})
``


---

## public/vite.svg
This is a svg file.

---

## src/assets/react.svg
This is a svg file.

