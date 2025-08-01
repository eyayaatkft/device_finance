from fastapi import APIRouter, UploadFile, File, HTTPException, Body, Query
from typing import Literal
from chat.models import ChatRequest
from chat.history import load_chat_history, save_chat_history
from chat.prompt import build_prompt, call_groq, extract_json_from_response, to_pascal_case, translate_with_gemini
from vectorstore.store import get_vector_store_for_url
from ingestion.file_ingest import ingest_file, remove_file, fetch_github_documents
from ingestion.url_ingest import ingest_url, remove_url
from utils.tracking import load_tracking
import shutil
import os
import logging
from dotenv import load_dotenv
import requests

load_dotenv()

router = APIRouter()

@router.get("/help-themes")
async def help_themes(url: str = Query(..., description="The URL to extract themes from")):
    try:
        if not url:
            return {"themes": [], "error": "URL parameter is required"}
        
        logging.info(f"[HelpThemes] Checking themes for URL: {url}")
        store = get_vector_store_for_url(url)
        
        # Debug: Check vector store status
        doc_count = store._collection.count()
        logging.info(f"[HelpThemes] Vector store document count: {doc_count}")
        
        sample = store._collection.get(limit=100)
        texts = sample.get('documents', [])
        logging.info(f"[HelpThemes] Sample texts count: {len(texts)}")
        
        if not texts:
            logging.info(f"[HelpThemes] No content found for URL: {url}")
            return {"themes": [],"message": "No knowledge base found for this repository. Please ingest the repo first."}
        content_sample = '\n'.join(texts[:20])
        prompt = f"""
Analyze the following support content. Identify up to 10 key themes or categories.
For each theme, provide:
- A short label (max 2 words)
- A Lucide React icon component name (e.g., ShoppingCart, CreditCard, Users)
- A concise, user-friendly snippet summarizing the theme
Return ONLY a JSON array of these objects, with no extra text, markdown, or explanations. Do not wrap in markdown or add any commentary.
Content sample:
{content_sample}
"""
        from groq import Groq
        client = Groq(api_key=os.getenv("GROQ_API_KEY"))
        response = client.chat.completions.create(
            model="llama3-8b-8192",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3,
            max_tokens=1024
        )
        raw = response.choices[0].message.content.strip()
        try:
            themes = extract_json_from_response(raw)
            for theme in themes:
                if 'icon' in theme:
                    theme['icon'] = to_pascal_case(theme['icon'])
            logging.info(f"Themes: {themes}")
        except Exception as e:
            logging.error(f"[HelpThemes] Failed to parse Groq response: {e}\nRaw: {raw}")
            themes = []
        return {"themes": themes}
    except Exception as e:
        logging.error(f"[HelpThemes Error] {e}")
        return {"themes": [], "error": str(e)}

@router.post("/chat")
async def chat(request: ChatRequest):
    question = request.question
    url = request.url
    user_lang = request.user_language or "en"
    user_id = request.user_id
    temperature = request.temperature
    max_output_tokens = request.max_output_tokens
    chat_history = load_chat_history(url, user_id)
    store = get_vector_store_for_url(url)
    print(f"[Chat] Received question: {question} for URL: {url}")
    print(f"[DEBUG] Vector store document count: {store._collection.count()}")
    sample = store._collection.get(limit=1)
    print(f"[DEBUG] Sample: {sample}")
    if not sample.get('documents'):
        return {"answer": "No knowledge base found for this repository. Please ingest the repo first.", "sources": []}
    docs_and_scores = store.similarity_search_with_score(question, k=4)
    docs = [doc for doc, score in docs_and_scores]
    logging.debug(f"[DEBUG] Question: {question}")
    logging.debug(f"[DEBUG] Top docs (first 100 chars each): {[doc.page_content[:100] for doc in docs]}")
    print(f"[DEBUG] User question: {question}")
    print(f"[DEBUG] User language: {user_lang}")# Translate chat history and question to English for LLM if needed
    if user_lang != "English":
        async def translate_turn(turn):
            return {
                "user": await translate_with_gemini(turn["user"], target_language="English"),
                "assistant": await translate_with_gemini(turn["assistant"], target_language="English")
            }
        chat_history_en = [await translate_turn(turn) for turn in chat_history]
        question_en = await translate_with_gemini(question, target_language="English")
    else:
        chat_history_en = chat_history[:5]
        question_en = question
    print(f"[DEBUG] User question: {question_en}")
    
    prompt = build_prompt(question_en, docs, chat_history_en)
    logging.debug(f"[DEBUG] Prompt (first 500 chars): {prompt[:500]}")
    answer_en = await call_groq(prompt, chat_history_en, temperature, max_output_tokens)
    print(answer_en)
    # Translate answer back to user's language if needed
    print(f"[DEBUG] User language: {user_lang}")
    if user_lang != "English":
        answer = await translate_with_gemini(answer_en, target_language=user_lang)
    else:
        answer = answer_en

    sources = [doc.metadata.get("source_file", "") for doc in docs]
    chat_history.append({"user": question, "assistant": answer})
    save_chat_history(url, user_id, chat_history)
    return {"answer": answer, "sources": sources}

@router.get("/chat/history")
def get_chat_history(url: str = Query(..., description="The URL to get chat history for"), 
                     user_id: str = Query(..., description="The user ID to get chat history for")):
    try:
        if not url or not user_id:
            return {"error": "Both url and user_id are required"}
        return load_chat_history(url, user_id)
    except Exception as e:
        logging.error(f"[Chat History Error] {e}")
        return {"error": str(e)}

@router.get("/knowledge/list")
def list_knowledge():
    try:
        tracking = load_tracking()
        return tracking
    except Exception as e:
        logging.error(f"[Knowledge List Error] {e}")
        return {"files": {}, "urls": {}, "error": str(e)}

@router.post("/knowledge/remove")
def remove_knowledge(item: str = Body(...), type: Literal["file", "url"] = Body(...)):
    if type == "file":
        remove_file(item)
    elif type == "url":
        remove_url(item)
    else:
        return {"success": False, "error": "Invalid type"}
    return {"success": True}

@router.post("/knowledge/reembed")
def reembed_knowledge(item: str = Body(...), type: Literal["file", "url"] = Body(...)):
    if type == "file":
        ingest_file(item)
    elif type == "url":
        # Note: ingest_url is async, but this endpoint is sync. Consider making async if needed.
        import asyncio
        asyncio.run(ingest_url(item))
    else:
        return {"success": False, "error": "Invalid type"}
    return {"success": True}

@router.post("/scrape-url")
async def scrape_url(url: str = Body(...), github_token: str = Body(None)):
    # Validate URL to prevent crawling large sites
    from urllib.parse import urlparse
    parsed = urlparse(url)
    
    # Prevent crawling of large domains
    large_domains = ['github.com', 'www.github.com', 'google.com', 'stackoverflow.com']
    if parsed.netloc in large_domains and not url.startswith('https://github.com/'):
        return {"success": False, "error": f"Cannot crawl {parsed.netloc}. Use specific repository URLs."}
    
    result = await ingest_url(url)
    return result

@router.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    try:
        data_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), '../../data'))
        dest_path = os.path.join(data_dir, file.filename)
        with open(dest_path, "wb") as buffer:
            shutil.copyfileobj(file.file, buffer)
        logging.info(f"[Upload] Saved file to {dest_path}")
        ingest_file(dest_path)
        return {"success": True, "filename": file.filename}
    except Exception as e:
        logging.error(f"[Upload Error] {e}")
        return {"success": False, "error": str(e)}
    


@router.post("/ingest-github")
async def ingest_github_repo(request: dict = Body(...)):
    """
    Ingest all supported documents from a GitHub repository.
    Uses the hash of the repo URL as the vector store name (multi-tenant).
    """
    try:
        # Log the incoming request for debugging
        logging.info(f"[GitHub Ingest] Received request: {request}")
        
        # Extract parameters from request dict
        github_url = request.get("github_url") or request.get("url")  # Handle both parameter names
        github_token = request.get("github_token")
        
        logging.info(f"[GitHub Ingest] Extracted URL: {github_url}, Token provided: {bool(github_token)}")
        
        if not github_url:
            return {"success": False, "error": "github_url or url is required"}
        
        # Validate GitHub URL
        if not github_url.startswith("https://github.com/"):
            return {"success": False, "error": "Invalid GitHub URL. Must start with https://github.com/"}
        
        # Remove .git if present
        if github_url.endswith('.git'):
            github_url = github_url[:-4]
        
        logging.info(f"[GitHub Ingest] Starting ingestion for {github_url}")
        result = await fetch_github_documents(github_url, github_token=github_token)
        
        logging.info(f"[GitHub Ingest] Completed with result: {result}")
        
        return {
            "success": True, 
            "repo": github_url,
            "files_processed": result.get("files_processed", 0),
            "files_ingested": result.get("files_ingested", 0),
            "message": f"Successfully ingested {result.get('files_ingested', 0)} files from {github_url}"
        }
    except ValueError as e:
        logging.error(f"[GitHub Ingest] Validation error: {e}")
        return {"success": False, "error": str(e)}
    except requests.exceptions.RequestException as e:
        logging.error(f"[GitHub Ingest] API error: {e}")
        return {"success": False, "error": f"GitHub API error: {str(e)}"}
    except Exception as e:
        logging.error(f"[GitHub Ingest] Unexpected error: {e}")
        return {"success": False, "error": f"Unexpected error: {str(e)}"}    
