from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from datetime import datetime
from typing import Optional
import asyncio
import httpx
from bs4 import BeautifulSoup
from playwright.async_api import async_playwright
import random

app = FastAPI(title="VC News Scraper API")

# Enable CORS for Lovable frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# User agents for rotation
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.2 Safari/605.1.15",
]

# Source configurations
SOURCES = {
    "techcrunch": {
        "name": "TechCrunch",
        "url": "https://techcrunch.com/category/venture/",
        "requires_js": False,
        "selectors": {
            "articles": "article.post-block",
            "title": "h2.post-block__title a",
            "author": "span.river-byline__authors a",
            "date": "time",
            "excerpt": "div.post-block__content",
            "link": "h2.post-block__title a"
        }
    },
    "a16z": {
        "name": "a16z",
        "url": "https://a16z.com/news-content/",
        "requires_js": True,
        "selectors": {
            "articles": "article, div.post-card, div[class*='card']",
            "title": "h2 a, h3 a, a.post-title",
            "author": "span.author, div.author",
            "date": "time, span.date",
            "excerpt": "p.excerpt, div.excerpt",
            "link": "h2 a, h3 a, a.post-title"
        }
    },
    "sequoia": {
        "name": "Sequoia",
        "url": "https://www.sequoiacap.com/articles/",
        "requires_js": True,
        "selectors": {
            "articles": "article, div.article-card, a[href*='/article']",
            "title": "h2, h3, span.title",
            "author": "span.author",
            "date": "time, span.date",
            "excerpt": "p.excerpt, p.summary",
            "link": "a"
        }
    },
    "firstround": {
        "name": "First Round Review",
        "url": "https://review.firstround.com/",
        "requires_js": False,
        "selectors": {
            "articles": "article, div.post, div.article-preview",
            "title": "h2 a, h3 a, a.title",
            "author": "span.author, p.author",
            "date": "time, span.date",
            "excerpt": "p.excerpt, p.description",
            "link": "h2 a, h3 a, a.title, a"
        }
    },
    "ycombinator": {
        "name": "Y Combinator",
        "url": "https://www.ycombinator.com/blog/",
        "requires_js": True,
        "selectors": {
            "articles": "article, div.post, a[href*='/blog/']",
            "title": "h2, h3, span.title",
            "author": "span.author",
            "date": "time, span.date",
            "excerpt": "p.excerpt, p",
            "link": "a"
        }
    },
    "crunchbase": {
        "name": "Crunchbase News",
        "url": "https://news.crunchbase.com/",
        "requires_js": False,
        "selectors": {
            "articles": "article, div.post-item",
            "title": "h2 a, h3 a, a.entry-title",
            "author": "span.author a, a.author",
            "date": "time, span.date",
            "excerpt": "p.excerpt, div.entry-summary p",
            "link": "h2 a, h3 a, a.entry-title"
        }
    }
}


class Article(BaseModel):
    title: str
    author: Optional[str] = None
    date: Optional[str] = None
    url: str
    excerpt: Optional[str] = None


class ScrapeResponse(BaseModel):
    success: bool
    source: str
    source_name: str
    scraped_at: str
    articles: list[Article]
    error: Optional[str] = None


def get_random_headers():
    return {
        "User-Agent": random.choice(USER_AGENTS),
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
        "Accept-Language": "en-US,en;q=0.5",
        "Accept-Encoding": "gzip, deflate, br",
        "Connection": "keep-alive",
    }


async def scrape_with_httpx(url: str) -> str:
    """Scrape static pages with httpx"""
    async with httpx.AsyncClient(follow_redirects=True, timeout=30.0) as client:
        response = await client.get(url, headers=get_random_headers())
        response.raise_for_status()
        return response.text


async def scrape_with_playwright(url: str) -> str:
    """Scrape JS-rendered pages with Playwright"""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        context = await browser.new_context(
            user_agent=random.choice(USER_AGENTS),
            viewport={"width": 1920, "height": 1080}
        )
        page = await context.new_page()
        
        try:
            await page.goto(url, wait_until="networkidle", timeout=30000)
            await page.wait_for_timeout(2000)  # Extra wait for dynamic content
            content = await page.content()
        finally:
            await browser.close()
        
        return content


def extract_text(element, selector: str) -> Optional[str]:
    """Safely extract text from an element"""
    if not element:
        return None
    
    # Try multiple selectors separated by comma
    for sel in selector.split(","):
        sel = sel.strip()
        found = element.select_one(sel)
        if found:
            return found.get_text(strip=True)
    
    return None


def extract_link(element, selector: str, base_url: str) -> Optional[str]:
    """Safely extract link from an element"""
    if not element:
        return None
    
    for sel in selector.split(","):
        sel = sel.strip()
        found = element.select_one(sel)
        if found:
            href = found.get("href")
            if href:
                if href.startswith("/"):
                    # Convert relative to absolute URL
                    from urllib.parse import urljoin
                    return urljoin(base_url, href)
                return href
    
    # If element itself is a link
    if element.name == "a":
        href = element.get("href")
        if href:
            if href.startswith("/"):
                from urllib.parse import urljoin
                return urljoin(base_url, href)
            return href
    
    return None


def parse_articles(html: str, config: dict, base_url: str) -> list[Article]:
    """Parse articles from HTML using configured selectors"""
    soup = BeautifulSoup(html, "html.parser")
    articles = []
    selectors = config["selectors"]
    
    # Find all article containers
    containers = soup.select(selectors["articles"])[:20]  # Limit to 20 articles
    
    for container in containers:
        title = extract_text(container, selectors["title"])
        if not title:
            continue  # Skip if no title
        
        link = extract_link(container, selectors["link"], base_url)
        if not link:
            continue  # Skip if no link
        
        article = Article(
            title=title,
            author=extract_text(container, selectors.get("author", "")),
            date=extract_text(container, selectors.get("date", "")),
            url=link,
            excerpt=extract_text(container, selectors.get("excerpt", ""))
        )
        
        # Truncate excerpt if too long
        if article.excerpt and len(article.excerpt) > 300:
            article.excerpt = article.excerpt[:297] + "..."
        
        articles.append(article)
    
    return articles


async def scrape_source(source_key: str) -> ScrapeResponse:
    """Scrape a single source"""
    if source_key not in SOURCES:
        return ScrapeResponse(
            success=False,
            source=source_key,
            source_name="Unknown",
            scraped_at=datetime.utcnow().isoformat() + "Z",
            articles=[],
            error=f"Unknown source: {source_key}"
        )
    
    config = SOURCES[source_key]
    
    try:
        # Add random delay to avoid rate limiting
        await asyncio.sleep(random.uniform(0.5, 1.5))
        
        # Scrape based on whether JS is required
        if config["requires_js"]:
            html = await scrape_with_playwright(config["url"])
        else:
            html = await scrape_with_httpx(config["url"])
        
        # Parse articles
        articles = parse_articles(html, config, config["url"])
        
        return ScrapeResponse(
            success=True,
            source=source_key,
            source_name=config["name"],
            scraped_at=datetime.utcnow().isoformat() + "Z",
            articles=articles
        )
    
    except Exception as e:
        return ScrapeResponse(
            success=False,
            source=source_key,
            source_name=config["name"],
            scraped_at=datetime.utcnow().isoformat() + "Z",
            articles=[],
            error=str(e)
        )


# API Endpoints

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat() + "Z"}


@app.get("/sources")
async def list_sources():
    """List all available sources"""
    return {
        "sources": [
            {"key": key, "name": config["name"], "url": config["url"]}
            for key, config in SOURCES.items()
        ]
    }


@app.get("/scrape/{source}", response_model=ScrapeResponse)
async def scrape_single_source(source: str):
    """Scrape a single source"""
    return await scrape_source(source)


@app.get("/scrape")
async def scrape_all_sources():
    """Scrape all sources and return combined results"""
    results = []
    all_articles = []
    
    # Scrape all sources concurrently (with some throttling)
    tasks = [scrape_source(source_key) for source_key in SOURCES.keys()]
    responses = await asyncio.gather(*tasks, return_exceptions=True)
    
    for response in responses:
        if isinstance(response, Exception):
            continue
        results.append({
            "source": response.source,
            "source_name": response.source_name,
            "success": response.success,
            "article_count": len(response.articles),
            "error": response.error
        })
        
        # Add source info to each article
        for article in response.articles:
            all_articles.append({
                "source": response.source,
                "source_name": response.source_name,
                **article.model_dump()
            })
    
    # Sort by date if available (newest first)
    # Articles without dates go to the end
    def sort_key(a):
        if a.get("date"):
            return (0, a["date"])
        return (1, "")
    
    all_articles.sort(key=sort_key, reverse=True)
    
    return {
        "success": True,
        "scraped_at": datetime.utcnow().isoformat() + "Z",
        "sources": results,
        "total_articles": len(all_articles),
        "articles": all_articles
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
