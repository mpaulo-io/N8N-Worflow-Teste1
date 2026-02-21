{
  "nodes": [
    {
      "parameters": {
        "model": "llama-3.3-70b-versatile",
        "options": {
          "maxTokensToSample": 6000,
          "temperature": 0.6
        }
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatGroq",
      "typeVersion": 1,
      "position": [1200, 304],
      "id": "groq-model-placeholder",
      "name": "Groq Chat Model",
      "credentials": {
        "groqApi": {
          "id": "{{CREDENTIAL_ID_GROQ}}",
          "name": "Groq_API_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://seusite-exemplo.com.br/wp-json/wp/v2/media",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpBasicAuth",
        "sendBody": true,
        "contentType": "multipart-form-data",
        "bodyParameters": {
          "parameters": [
            {
              "parameterType": "formBinaryData",
              "name": "file",
              "inputDataFieldName": "data"
            },
            {
              "name": "alt_text",
              "value": "={{ $json.alt_texto || $json.featured_image_alt || $('Basic LLM Chain').item.json.alt_text || \"criança feliz aprendendo com mascote educativo, estilo 3D amigável\" }}"
            },
            {
              "name": "title",
              "value": "={{ $json.title || \"Imagem do Post - Projeto Portfolio\" }}"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [656, -80],
      "id": "upload-wp-placeholder",
      "name": "Upload Imagem WP",
      "credentials": {
        "httpBasicAuth": {
          "id": "{{CREDENTIAL_ID_WP}}",
          "name": "WordPress_BasicAuth_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "documentId": {
          "__rl": true,
          "value": "{{GOOGLE_SHEET_ID}}",
          "mode": "list",
          "cachedResultName": "Editorial Calendar - Portfolio",
          "cachedResultUrl": "https://docs.google.com/spreadsheets/d/{{GOOGLE_SHEET_ID}}/edit"
        },
        "sheetName": {
          "__rl": true,
          "value": "gid=0",
          "mode": "list",
          "cachedResultName": "Topics",
          "cachedResultUrl": "https://docs.google.com/spreadsheets/d/{{GOOGLE_SHEET_ID}}/edit#gid=0"
        },
        "filtersUI": {
          "values": [
            {
              "lookupColumn": "Status",
              "lookupValue": "Pendente"
            }
          ]
        },
        "options": {
          "returnFirstMatch": true
        }
      },
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.7,
      "position": [640, 112],
      "id": "sheets-read-placeholder",
      "name": "1. Buscar Tema na Planilha",
      "credentials": {
        "googleSheetsOAuth2Api": {
          "id": "{{CREDENTIAL_ID_GS}}",
          "name": "GoogleSheets_OAuth_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "documentId": {
          "__rl": true,
          "value": "{{GOOGLE_SHEET_ID}}",
          "mode": "list",
          "cachedResultName": "Editorial Calendar - Portfolio",
          "cachedResultUrl": "https://docs.google.com/spreadsheets/d/{{GOOGLE_SHEET_ID}}/edit"
        },
        "sheetName": {
          "__rl": true,
          "value": "gid=1",
          "mode": "list",
          "cachedResultName": "Affiliates",
          "cachedResultUrl": "https://docs.google.com/spreadsheets/d/{{GOOGLE_SHEET_ID}}/edit#gid=1"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.7,
      "position": [800, 112],
      "id": "sheets-affiliate-placeholder",
      "name": "2. Buscar Produto Afiliado",
      "credentials": {
        "googleSheetsOAuth2Api": {
          "id": "{{CREDENTIAL_ID_GS}}",
          "name": "GoogleSheets_OAuth_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "aggregate": "aggregateAllItemData",
        "options": {}
      },
      "type": "n8n-nodes-base.aggregate",
      "typeVersion": 1,
      "position": [992, 112],
      "id": "aggregate-placeholder",
      "name": "3. Agregar Dados da Planilha"
    },
    {
      "parameters": {
        "jsCode": "// ============ 1. CAPTURA SEGURA DOS DADOS ============\nconst llmNode = $('Basic LLM Chain');\nconst planilhaNode = $('1. Buscar Tema na Planilha');\nconst uploadNode = $('Upload Imagem WP');\n\nconst llmRaw = llmNode?.item?.json?.text || llmNode?.item?.json?.output || \"\";\nconst planilha = planilhaNode?.item?.json || {};\nconst imageId = uploadNode?.item?.json?.id ? parseInt(uploadNode.item.json.id, 10) : null;\n\nconst fallbackTema = planilha[\"Tema\"] || \"Novo Post - Projeto Portfolio\";\nconst fallbackKeyword = (planilha[\"Palavra-Chave\"] || \"\").trim();\n\n// ============ 2. PARSE INTELIGENTE DO JSON ============\nlet artigo = {};\nfunction tentarParse(texto) {\n    if (!texto || typeof texto !== 'string') return null;\n    try {\n        let clean = texto.trim()\n            .replace(/^```json\\s*/gi, \"\")\n            .replace(/```\\s*$/gi, \"\")\n            .replace(/^\\{/, \"{\")\n            .replace(/\\}$/, \"}\")\n            .trim();\n        const start = clean.indexOf('{');\n        const end = clean.lastIndexOf('}');\n        if (start !== -1 && end !== -1) {\n            const jsonStr = clean.substring(start, end + 1);\n            return JSON.parse(jsonStr);\n        }\n    } catch (e) {\n        console.warn('Parse JSON falhou:', e.message);\n    }\n    return null;\n}\nartigo = tentarParse(llmRaw) || {};\n\n// ============ 3. EXTRAÇÃO E LIMPEZA DE HTML ============\nlet finalHtml = artigo.content_html || artigo.content || \"\";\nif (!finalHtml && llmRaw) {\n    const match = llmRaw.match(/\"content_html\"\\s*:\\s*\"([\\s\\S]*?)\"\\s*[,}]/i);\n    if (match && match[1]) finalHtml = match[1];\n}\n\n// Decode de entidades HTML\nfinalHtml = finalHtml\n    .replace(/&quot;/g, '\"').replace(/&amp;/g, '&').replace(/&lt;/g, '<')\n    .replace(/&gt;/g, '>').replace(/&#39;/g, \"'\").replace(/&nbsp;/g, ' ')\n    .replace(/&[a-zA-Z]+;/g, '');\n\n// Sanitização final\nfinalHtml = finalHtml\n    .replace(/\\\\\"/g, '\"').replace(/\\\\'/g, \"'\").replace(/\\\\n/g, '\\n').replace(/\\\\\\\\/g, '\\\\')\n    .replace(/<script[^>]*>[\\s\\S]*?<\\/script>/gi, '')\n    .replace(/<h1[^>]*>[\\s\\S]*?<\\/h1>/gi, '')\n    .replace(/\\s{2,}/g, ' ').trim();\n\n// Validação crítica\nif (!finalHtml || finalHtml.length < 300) {\n    finalHtml = `<div style=\"background:#fff3cd;border:1px solid #ffeaa7;padding:15px;border-radius:8px;\"><strong>⚠️ Atenção:</strong> Conteúdo gerado pela IA estava incompleto. Revise antes de publicar.</div><p><strong>${fallbackKeyword}</strong> é uma oportunidade para educar de forma leve e prazerosa.</p><p>Use recursos visuais, músicas e brinque junto. O segredo está na conexão emocional.</p><p><em>Conteúdo mínimo gerado automaticamente.</em></p>`;\n}\n\n// Metadados SEO\nconst title = (artigo.title || fallbackTema).substring(0, 200);\nconst safeSlug = (artigo.slug || title).toLowerCase().normalize('NFD').replace(/[\\u0300-\\u036f]/g, '').replace(/[^a-z0-9\\s-]/g, '').replace(/\\s+/g, '-').replace(/-+/g, '-').substring(0, 200);\n\nlet metaDesc = artigo.meta_description || \"\";\nif (!metaDesc || metaDesc.length < 50) {\n    const plainText = finalHtml.replace(/<[^>]+>/g, \" \").replace(/\\s{2,}/g, \" \").trim();\n    metaDesc = plainText.substring(0, 155) + (plainText.length > 155 ? \"...\" : \"\");\n}\n\n// FAQ Schema\nlet faqs = [];\nif (artigo.faqs && Array.isArray(artigo.faqs)) {\n    faqs = artigo.faqs.filter(f => f?.q && f?.a).map(f => ({\n        \"@type\": \"Question\",\n        \"name\": String(f.q).substring(0, 200),\n        \"acceptedAnswer\": { \"@type\": \"Answer\", \"text\": String(f.a).substring(0, 1000) }\n    })).slice(0, 10);\n}\n\n// Retorno final\nreturn [{\n    json: {\n        title: title,\n        slug: safeSlug,\n        content: finalHtml,\n        status: \"draft\",\n        featured_media: imageId,\n        rank_math_focus_keyword: fallbackKeyword,\n        rank_math_title: title,\n        rank_math_description: metaDesc,\n        rank_math_faq_schema: faqs.length > 0 ? JSON.stringify(faqs) : \"\",\n        _debug_content_length: finalHtml.length,\n        _debug_parse_success: !!artigo.content_html,\n        _generated_at: new Date().toISOString()\n    }\n}];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [928, -80],
      "id": "code-prepare-post-placeholder",
      "name": "7. Preparar Body para Post"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://seusite-exemplo.com.br/wp-json/wp/v2/posts",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpBasicAuth",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            { "name": "title", "value": "={{ $json.title }}" },
            { "name": "content", "value": "={{ $json.content }}" },
            { "name": "slug", "value": "={{ $json.slug }}" },
            { "name": "status", "value": "draft" },
            { "name": "featured_media", "value": "={{ $json.featured_media }}" }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [1200, -80],
      "id": "wp-create-post-placeholder",
      "name": "9. Criar Post Draft no WP",
      "credentials": {
        "httpBasicAuth": {
          "id": "{{CREDENTIAL_ID_WP}}",
          "name": "WordPress_BasicAuth_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "rule": {
          "interval": [
            { "triggerAtHour": 8, "triggerAtMinute": 15 },
            { "triggerAtHour": 14, "triggerAtMinute": 30 },
            { "triggerAtHour": 22, "triggerAtMinute": 22 }
          ]
        }
      },
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.3,
      "position": [448, 112],
      "id": "schedule-trigger-placeholder",
      "name": "Schedule Trigger"
    },
    {
      "parameters": {
        "operation": "update",
        "documentId": {
          "__rl": true,
          "value": "{{GOOGLE_SHEET_ID}}",
          "mode": "list",
          "cachedResultName": "Editorial Calendar - Portfolio",
          "cachedResultUrl": "https://docs.google.com/spreadsheets/d/{{GOOGLE_SHEET_ID}}/edit"
        },
        "sheetName": {
          "__rl": true,
          "value": "gid=0",
          "mode": "list",
          "cachedResultName": "Topics",
          "cachedResultUrl": "https://docs.google.com/spreadsheets/d/{{GOOGLE_SHEET_ID}}/edit#gid=0"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "row_number": "={{ $node[\"1. Buscar Tema na Planilha\"].json[\"row_number\"] }}",
            "Status": "Publicado"
          },
          "matchingColumns": ["row_number"],
          "schema": [
            { "id": "Status", "displayName": "Status", "type": "string", "canBeUsedToMatch": true },
            { "id": "row_number", "displayName": "row_number", "type": "number", "canBeUsedToMatch": true }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.7,
      "position": [1856, -80],
      "id": "sheets-update-placeholder",
      "name": "11. Marcar como Postado na Planilha",
      "credentials": {
        "googleSheetsOAuth2Api": {
          "id": "{{CREDENTIAL_ID_GS}}",
          "name": "GoogleSheets_OAuth_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Você é [Persona Especialista], profissional com experiência em educação infantil e ensino bilíngue. Escreve com autoridade, carinho genuíno e exemplos práticos. Tom acolhedor e direto.\n\nDADOS DE ENTRADA:\n• Tema: {{ $('1. Buscar Tema na Planilha').item.json[\"Tema\"] }}\n• Palavra-chave SEO: {{ $('1. Buscar Tema na Planilha').item.json[\"Palavra-Chave\"] }}\n• Produto Afiliado: {{ $('3. Agregar Dados da Planilha').item.json[\"data\"][0][\"Nome do Produto\"] }}\n• Link: {{ $('3. Agregar Dados da Planilha').item.json[\"data\"][0][\"Link de Afiliado\"] }}\n\nMISSÃO: Criar artigo PILLAR (1000-1500 palavras) que:\n1. Eduque sobre o tema\n2. Seja otimizado para SEO\n3. Inclua link afiliado naturalmente\n4. Tenha estrutura pedagógica\n\nESTRUTURA:\n1. INTRO: História/gancho com palavra-chave na primeira frase\n2. CORPO: 5-8 seções H2/H3 com teoria + prática + storytelling\n3. ELEMENTOS: <ul> com dicas, tabela educativa, links contextualizados, <strong> em keywords\n\nTABELA EDUCATIVA (formato obrigatório):\n<table style=\"width:100%; border-collapse:collapse; margin:30px 0;\"><thead><tr style=\"background:#667eea; color:white;\"><th style=\"border:1px solid #e0e0e0; padding:14px;\">🇬🇧 Expressão</th><th style=\"border:1px solid #e0e0e0; padding:14px;\">🇧🇷 Tradução</th><th style=\"border:1px solid #e0e0e0; padding:14px;\">🗣️ Pronúncia</th></tr></thead><tbody><tr><td style=\"border:1px solid #e0e0e0; padding:12px;\"><strong>Hello!</strong></td><td style=\"border:1px solid #e0e0e0; padding:12px;\">Olá!</td><td style=\"border:1px solid #e0e0e0; padding:12px;\"><em>He-lou</em></td></tr><!-- ADICIONE LINHAS RELEVANTES AO TEMA --></tbody></table>\n\nLINKS:\n- Afiliado: <a href=\"{{ $('3. Agregar Dados da Planilha').item.json[\"data\"][0][\"Link de Afiliado\"] }}\" target=\"_blank\" rel=\"noopener\">{{ $('3. Agregar Dados da Planilha').item.json[\"data\"][0][\"Nome do Produto\"] }}</a>\n- Educativo: <a href=\"https://exemplo-educativo.com\" target=\"_blank\" rel=\"noopener\">fonte educativa</a>\n\nREGRAS HTML:\n- <p> em todos parágrafos, <h2>/<h3> para seções, <strong> em keywords\n- NUNCA use <h1>, Markdown, ou escape de aspas\n\nSEO:\n- Palavra-chave: 10-15x (densidade 1-1.5%)\n- Título: palavra-chave + gancho (max 60 chars)\n- Meta description: problema + solução + CTA (max 155 chars)\n\nRESPOSTA (JSON PURO, sem explicações):\n{\n  \"title\": \"Título com palavra-chave + gancho (max 60 chars)\",\n  \"slug\": \"url-amigavel-sem-acentos\",\n  \"meta_description\": \"Resumo com palavra-chave (max 155 chars)\",\n  \"featured_image_alt\": \"[palavra-chave] - ilustração educativa estilo 3D amigável\",\n  \"content_html\": \"<p>Conteúdo HTML completo aqui</p>\"\n}\n\nGere o artigo. Responda APENAS o JSON.",
        "batching": {}
      },
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1.9,
      "position": [1408, 112],
      "id": "llm-chain-placeholder",
      "name": "Basic LLM Chain"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://seusite-exemplo.com.br/wp-json/custom/v1/update-rankmath",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpBasicAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"post_id\": {{ $node[\"9. Criar Post Draft no WP\"].json.id }},\n  \"rank_math_focus_keyword\": \"{{ $node[\"7. Preparar Body para Post\"].json.rank_math_focus_keyword }}\",\n  \"rank_math_title\": \"{{ $node[\"7. Preparar Body para Post\"].json.rank_math_title }}\",\n  \"rank_math_description\": \"{{ $node[\"7. Preparar Body para Post\"].json.rank_math_description }}\"\n}",
        "options": {}
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [1552, -96],
      "id": "rankmath-placeholder",
      "name": "Refresh Rank Math",
      "credentials": {
        "httpBasicAuth": {
          "id": "{{CREDENTIAL_ID_WP}}",
          "name": "WordPress_BasicAuth_Portfolio"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// Debug: Payload Rank Math\nconst payload = {\n  post_id: $node[\"9. Criar Post Draft no WP\"].json.id,\n  rank_math_focus_keyword: $node[\"7. Preparar Body para Post\"].json.rank_math_focus_keyword,\n  rank_math_title: $node[\"7. Preparar Body para Post\"].json.rank_math_title,\n  rank_math_description: $node[\"7. Preparar Body para Post\"].json.rank_math_description\n};\nconsole.log('Payload Rank Math:', JSON.stringify(payload, null, 2));\nif (!payload.rank_math_focus_keyword) {\n  throw new Error('rank_math_focus_keyword vazio! Verifique node anterior.');\n}\nreturn [{ json: payload }];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1376, -96],
      "id": "debug-rankmath-placeholder",
      "name": "Debug: Rank Math Payload"
    },
    {
      "parameters": {
        "jsCode": "// Montar URL Pollinations.ai (sem espaços, sem acentos)\nconst keyword = $('1. Buscar Tema na Planilha').item.json['Palavra-Chave'] || 'educational content';\nconst cleanKeyword = keyword.normalize('NFD').replace(/[\\u0300-\\u036f]/g, '').replace(/[^a-zA-Z0-9\\s]/g, '').replace(/\\s+/g, ' ').trim();\nconst basePrompt = 'educational mascot, friendly teacher character, 3d friendly style, ' + cleanKeyword + ', children learning, educational scene, warm lighting, ultra detailed, high quality, bright colors, family friendly';\nconst encodedPrompt = encodeURIComponent(basePrompt);\nconst seed = Math.floor(Math.random() * 100000);\nconst imageUrl = `https://image.pollinations.ai/prompt/${encodedPrompt}?width=1080&height=720&nologo=true&seed=${seed}`;\nconsole.log('🎨 Prompt:', cleanKeyword);\nconsole.log('🔗 URL:', imageUrl);\nreturn [{ json: { imageUrl, prompt: cleanKeyword, seed, _debug_url_length: imageUrl.length } }];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1680, 112],
      "id": "code-build-image-url-placeholder",
      "name": "4.5. Montar URL da Imagem"
    },
    {
      "parameters": {
        "method": "GET",
        "url": "{{ $json.imageUrl }}",
        "options": {
          "response": { "response": { "responseFormat": "file" }, "timeout": 30000 }
        }
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.3,
      "position": [1872, 112],
      "id": "pollinations-fetch-placeholder",
      "name": "5. Gerar Imagem Pollinations"
    }
  ],
  "connections": {
    "Groq Chat Model": { "ai_languageModel": [[{ "node": "Basic LLM Chain", "type": "ai_languageModel", "index": 0 }]] },
    "Upload Imagem WP": { "main": [[{ "node": "7. Preparar Body para Post", "type": "main", "index": 0 }]] },
    "1. Buscar Tema na Planilha": { "main": [[{ "node": "2. Buscar Produto Afiliado", "type": "main", "index": 0 }]] },
    "2. Buscar Produto Afiliado": { "main": [[{ "node": "3. Agregar Dados da Planilha", "type": "main", "index": 0 }]] },
    "3. Agregar Dados da Planilha": { "main": [[{ "node": "Basic LLM Chain", "type": "main", "index": 0 }]] },
    "7. Preparar Body para Post": { "main": [[{ "node": "9. Criar Post Draft no WP", "type": "main", "index": 0 }]] },
    "9. Criar Post Draft no WP": { "main": [[{ "node": "Debug: Rank Math Payload", "type": "main", "index": 0 }]] },
    "Schedule Trigger": { "main": [[{ "node": "1. Buscar Tema na Planilha", "type": "main", "index": 0 }]] },
    "11. Marcar como Postado na Planilha": { "main": [[]] },
    "Basic LLM Chain": { "main": [[{ "node": "4.5. Montar URL da Imagem", "type": "main", "index": 0 }]] },
    "Refresh Rank Math": { "main": [[{ "node": "11. Marcar como Postado na Planilha", "type": "main", "index": 0 }]] },
    "Debug: Rank Math Payload": { "main": [[{ "node": "Refresh Rank Math", "type": "main", "index": 0 }]] },
    "4.5. Montar URL da Imagem": { "main": [[{ "node": "5. Gerar Imagem Pollinations", "type": "main", "index": 0 }]] },
    "5. Gerar Imagem Pollinations": { "main": [[{ "node": "Upload Imagem WP", "type": "main", "index": 0 }]] }
  },
  "pinData": {},
  "meta": {
    "templateCredsSetupCompleted": false,
    "instanceId": "portfolio-demo-instance",
    "description": "Workflow de automação de conteúdo com IA - Versão Portfolio (dados fictícios)"
  }
}
