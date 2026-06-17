# AWS Backend Implementation Guide - Prompt 3

Complete code examples and setup instructions for Amazon RDS PostgreSQL and FastAPI WebSocket progress tracking.

## 1. Dependencies

Add to `requirements.txt`:
```
sqlalchemy>=2.0.0
asyncpg>=0.28.0
psycopg2-binary>=2.9.0
alembic>=1.12.0
```

Install:
```bash
pip install sqlalchemy asyncpg psycopg2-binary alembic
```

## 2. Database Models

### `backend/models.py`
```python
from sqlalchemy import Column, Integer, String, DateTime, Float, JSON, ForeignKey, Enum
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime
import enum

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    password_hash = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    analyses = relationship("Analysis", back_populates="user")

class Analysis(Base):
    __tablename__ = "analyses"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    aws_account_id = Column(String, index=True)
    aws_region = Column(String)
    resources_scanned = Column(Integer, default=0)
    issues_found = Column(Integer, default=0)
    estimated_monthly_savings = Column(Float, default=0)
    estimated_annual_savings = Column(Float, default=0)
    analysis_result = Column(JSON)
    status = Column(String, default="completed")  # pending, running, completed, failed
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    
    user = relationship("User", back_populates="analyses")
    resources = relationship("Resource", back_populates="analysis")
    recommendations = relationship("Recommendation", back_populates="analysis")

class Resource(Base):
    __tablename__ = "resources"
    
    id = Column(Integer, primary_key=True, index=True)
    analysis_id = Column(Integer, ForeignKey("analyses.id"))
    resource_type = Column(String)  # ec2, rds, s3, lambda, ebs, elb, etc.
    resource_id = Column(String)
    resource_name = Column(String)
    resource_config = Column(JSON)
    cost_estimate = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    analysis = relationship("Analysis", back_populates="resources")

class Recommendation(Base):
    __tablename__ = "recommendations"
    
    id = Column(Integer, primary_key=True, index=True)
    analysis_id = Column(Integer, ForeignKey("analyses.id"))
    severity = Column(String)  # high, medium, low
    title = Column(String)
    description = Column(String)
    estimated_savings = Column(Float)
    fix_command = Column(String)
    aws_cli_command = Column(String)
    terraform_code = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    analysis = relationship("Analysis", back_populates="recommendations")
```

## 3. Database Connection

### `backend/db.py`
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.pool import NullPool
import os
from models import Base
from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")

if not DATABASE_URL:
    raise ValueError("DATABASE_URL not set in environment variables")

# Convert PostgreSQL URL to async format
if DATABASE_URL.startswith("postgresql://"):
    DATABASE_URL = DATABASE_URL.replace("postgresql://", "postgresql+asyncpg://", 1)

# Create async engine
engine = create_async_engine(
    DATABASE_URL,
    echo=False,
    poolclass=NullPool,
    pool_pre_ping=True
)

# Create session factory
AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

async def init_db():
    """Initialize database tables"""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def get_db() -> AsyncSession:
    """Get database session"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

## 4. Database Service

### `backend/db_service.py`
```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.future import select
from models import User, Analysis, Resource, Recommendation
from db import AsyncSessionLocal
from datetime import datetime
from typing import List, Optional
import json

class DatabaseService:
    def __init__(self):
        pass
    
    async def create_user(self, email: str, password_hash: str) -> User:
        """Create new user"""
        async with AsyncSessionLocal() as session:
            user = User(email=email, password_hash=password_hash)
            session.add(user)
            await session.commit()
            await session.refresh(user)
            return user
    
    async def get_user_by_email(self, email: str) -> Optional[User]:
        """Get user by email"""
        async with AsyncSessionLocal() as session:
            result = await session.execute(
                select(User).where(User.email == email)
            )
            return result.scalars().first()
    
    async def create_analysis(self, user_id: int, account_id: str, region: str) -> Analysis:
        """Create new analysis record"""
        async with AsyncSessionLocal() as session:
            analysis = Analysis(
                user_id=user_id,
                aws_account_id=account_id,
                aws_region=region,
                status="running"
            )
            session.add(analysis)
            await session.commit()
            await session.refresh(analysis)
            return analysis
    
    async def update_analysis(
        self,
        analysis_id: int,
        resources_scanned: int = None,
        issues_found: int = None,
        estimated_monthly_savings: float = None,
        estimated_annual_savings: float = None,
        analysis_result: dict = None,
        status: str = None
    ) -> Analysis:
        """Update analysis record"""
        async with AsyncSessionLocal() as session:
            result = await session.execute(
                select(Analysis).where(Analysis.id == analysis_id)
            )
            analysis = result.scalars().first()
            
            if not analysis:
                raise ValueError(f"Analysis {analysis_id} not found")
            
            if resources_scanned is not None:
                analysis.resources_scanned = resources_scanned
            if issues_found is not None:
                analysis.issues_found = issues_found
            if estimated_monthly_savings is not None:
                analysis.estimated_monthly_savings = estimated_monthly_savings
            if estimated_annual_savings is not None:
                analysis.estimated_annual_savings = estimated_annual_savings
            if analysis_result is not None:
                analysis.analysis_result = analysis_result
            if status is not None:
                analysis.status = status
            
            await session.commit()
            await session.refresh(analysis)
            return analysis
    
    async def add_resource(
        self,
        analysis_id: int,
        resource_type: str,
        resource_id: str,
        resource_name: str,
        resource_config: dict,
        cost_estimate: float
    ) -> Resource:
        """Add resource to analysis"""
        async with AsyncSessionLocal() as session:
            resource = Resource(
                analysis_id=analysis_id,
                resource_type=resource_type,
                resource_id=resource_id,
                resource_name=resource_name,
                resource_config=resource_config,
                cost_estimate=cost_estimate
            )
            session.add(resource)
            await session.commit()
            await session.refresh(resource)
            return resource
    
    async def add_recommendation(
        self,
        analysis_id: int,
        severity: str,
        title: str,
        description: str,
        estimated_savings: float,
        fix_command: str = None,
        aws_cli_command: str = None,
        terraform_code: str = None
    ) -> Recommendation:
        """Add recommendation to analysis"""
        async with AsyncSessionLocal() as session:
            recommendation = Recommendation(
                analysis_id=analysis_id,
                severity=severity,
                title=title,
                description=description,
                estimated_savings=estimated_savings,
                fix_command=fix_command,
                aws_cli_command=aws_cli_command,
                terraform_code=terraform_code
            )
            session.add(recommendation)
            await session.commit()
            await session.refresh(recommendation)
            return recommendation
    
    async def get_user_analyses(
        self,
        user_id: int,
        limit: int = 10,
        offset: int = 0
    ) -> List[Analysis]:
        """Get user's analyses with pagination"""
        async with AsyncSessionLocal() as session:
            result = await session.execute(
                select(Analysis)
                .where(Analysis.user_id == user_id)
                .order_by(Analysis.created_at.desc())
                .limit(limit)
                .offset(offset)
            )
            return result.scalars().all()
    
    async def get_analysis(self, analysis_id: int) -> Optional[Analysis]:
        """Get specific analysis"""
        async with AsyncSessionLocal() as session:
            result = await session.execute(
                select(Analysis).where(Analysis.id == analysis_id)
            )
            return result.scalars().first()
    
    async def get_analysis_resources(self, analysis_id: int) -> List[Resource]:
        """Get resources for an analysis"""
        async with AsyncSessionLocal() as session:
            result = await session.execute(
                select(Resource).where(Resource.analysis_id == analysis_id)
            )
            return result.scalars().all()
    
    async def get_analysis_recommendations(self, analysis_id: int) -> List[Recommendation]:
        """Get recommendations for an analysis"""
        async with AsyncSessionLocal() as session:
            result = await session.execute(
                select(Recommendation).where(Recommendation.analysis_id == analysis_id)
            )
            return result.scalars().all()
```

## 5. WebSocket Progress Tracking

### `backend/websocket_manager.py`
```python
from fastapi import WebSocket, WebSocketDisconnect
from typing import List, Dict
import json
from datetime import datetime
import asyncio

class ConnectionManager:
    def __init__(self):
        # Store active connections: {analysis_id: [websockets]}
        self.active_connections: Dict[str, List[WebSocket]] = {}
    
    async def connect(self, websocket: WebSocket, analysis_id: str):
        """Accept WebSocket connection"""
        await websocket.accept()
        
        if analysis_id not in self.active_connections:
            self.active_connections[analysis_id] = []
        
        self.active_connections[analysis_id].append(websocket)
        print(f"Client connected to analysis {analysis_id}")
    
    def disconnect(self, analysis_id: str, websocket: WebSocket):
        """Remove WebSocket connection"""
        if analysis_id in self.active_connections:
            self.active_connections[analysis_id].remove(websocket)
            
            if not self.active_connections[analysis_id]:
                del self.active_connections[analysis_id]
    
    async def broadcast_progress(self, analysis_id: str, message: Dict):
        """Broadcast progress to all connected clients"""
        message["timestamp"] = datetime.utcnow().isoformat()
        
        if analysis_id in self.active_connections:
            disconnected = []
            
            for connection in self.active_connections[analysis_id]:
                try:
                    await connection.send_json(message)
                except Exception as e:
                    print(f"Error sending message: {e}")
                    disconnected.append(connection)
            
            # Remove disconnected clients
            for conn in disconnected:
                self.disconnect(analysis_id, conn)

manager = ConnectionManager()
```

## 6. Updated Main Backend

### Update `backend/main.py`
```python
from fastapi import FastAPI, HTTPException, WebSocket, WebSocketDisconnect, Depends
from fastapi.middleware.cors import CORSMiddleware
from dotenv import load_dotenv
import os
from datetime import datetime
from db import init_db, get_db
from db_service import DatabaseService
from aws_scanner import AWSScanner
from ai_analyzer import AIAnalyzer
from websocket_manager import manager
import asyncio

load_dotenv()

app = FastAPI(title="AI Cloud Cost Detective", version="1.0.0")

# Enable CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize services
aws_scanner = AWSScanner()
ai_analyzer = AIAnalyzer()
db_service = DatabaseService()

@app.on_event("startup")
async def startup():
    """Initialize database on startup"""
    await init_db()

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "service": "AI Cloud Cost Detective"}

@app.websocket("/ws/progress/{analysis_id}")
async def websocket_progress(websocket: WebSocket, analysis_id: str):
    """WebSocket endpoint for real-time progress updates"""
    await manager.connect(websocket, analysis_id)
    
    try:
        while True:
            # Keep connection alive
            await websocket.receive_text()
    except WebSocketDisconnect:
        manager.disconnect(analysis_id, websocket)

@app.post("/api/analyze")
async def analyze(request: dict):
    """
    Analyze AWS resources with real-time WebSocket progress
    """
    try:
        user_id = request.get("user_id")
        account_id = request.get("account_id")
        region = request.get("region")
        
        if not user_id or not account_id or not region:
            raise HTTPException(status_code=400, detail="user_id, account_id, region required")
        
        # Create analysis record
        analysis = await db_service.create_analysis(user_id, account_id, region)
        analysis_id = analysis.id
        
        # Broadcast start event
        await manager.broadcast_progress(str(analysis_id), {
            "status": "started",
            "message": "Starting AWS resource analysis..."
        })
        
        # Scan resources
        await manager.broadcast_progress(str(analysis_id), {
            "status": "scanning",
            "step": "EC2",
            "message": "Fetching EC2 instances..."
        })
        resources = await aws_scanner.scan_resources(account_id, region)
        
        total_resources = sum(len(v) if isinstance(v, list) else 1 for v in resources.values())
        
        await manager.broadcast_progress(str(analysis_id), {
            "status": "scanned",
            "resources_found": total_resources,
            "message": f"Scanned {total_resources} resources"
        })
        
        # Analyze with AI
        await manager.broadcast_progress(str(analysis_id), {
            "status": "analyzing",
            "message": "Running AI cost analysis..."
        })
        
        analysis_result = await ai_analyzer.analyze(resources)
        
        # Store results
        await manager.broadcast_progress(str(analysis_id), {
            "status": "storing",
            "message": "Storing analysis results..."
        })
        
        await db_service.update_analysis(
            analysis_id,
            resources_scanned=total_resources,
            issues_found=len(analysis_result.get("issues", [])),
            estimated_monthly_savings=analysis_result.get("estimated_monthly_savings", 0),
            estimated_annual_savings=analysis_result.get("estimated_annual_savings", 0),
            analysis_result=analysis_result,
            status="completed"
        )
        
        # Broadcast completion
        await manager.broadcast_progress(str(analysis_id), {
            "status": "completed",
            "message": "Analysis complete!",
            "estimated_monthly_savings": analysis_result.get("estimated_monthly_savings"),
            "estimated_annual_savings": analysis_result.get("estimated_annual_savings")
        })
        
        return {
            "analysis_id": analysis_id,
            "account_id": account_id,
            "region": region,
            "resources_scanned": total_resources,
            "analysis": analysis_result
        }
    
    except Exception as e:
        await manager.broadcast_progress(str(analysis.id if 'analysis' in locals() else '0'), {
            "status": "failed",
            "error": str(e)
        })
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/history")
async def get_history(user_id: int, limit: int = 10, offset: int = 0):
    """Get user's analysis history"""
    try:
        analyses = await db_service.get_user_analyses(user_id, limit, offset)
        
        return {
            "analyses": [
                {
                    "id": a.id,
                    "account_id": a.aws_account_id,
                    "region": a.aws_region,
                    "resources_scanned": a.resources_scanned,
                    "issues_found": a.issues_found,
                    "estimated_savings": a.estimated_monthly_savings,
                    "created_at": a.created_at.isoformat()
                }
                for a in analyses
            ]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/analyses/{analysis_id}")
async def get_analysis(analysis_id: int):
    """Get specific analysis details"""
    try:
        analysis = await db_service.get_analysis(analysis_id)
        
        if not analysis:
            raise HTTPException(status_code=404, detail="Analysis not found")
        
        resources = await db_service.get_analysis_resources(analysis_id)
        recommendations = await db_service.get_analysis_recommendations(analysis_id)
        
        return {
            "id": analysis.id,
            "account_id": analysis.aws_account_id,
            "region": analysis.aws_region,
            "resources_scanned": analysis.resources_scanned,
            "issues_found": analysis.issues_found,
            "estimated_monthly_savings": analysis.estimated_monthly_savings,
            "estimated_annual_savings": analysis.estimated_annual_savings,
            "analysis_result": analysis.analysis_result,
            "resources": resources,
            "recommendations": recommendations,
            "created_at": analysis.created_at.isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, reload=True)
```

## 7. RDS Setup Instructions

### Create RDS Instance
```bash
# Using AWS CLI
aws rds create-db-instance \
  --db-instance-identifier my-cost-detective-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username postgres \
  --master-user-password YourSecurePassword123! \
  --allocated-storage 20 \
  --publicly-accessible
```

### Get RDS Endpoint
```bash
aws rds describe-db-instances \
  --db-instance-identifier my-cost-detective-db \
  --query 'DBInstances[0].Endpoint.Address'
```

### Update .env
```env
DATABASE_URL=postgresql://postgres:YourSecurePassword123!@my-cost-detective-db.xxxxx.us-east-1.rds.amazonaws.com:5432/costdetective
```

## 8. Database Migrations (Optional)

### Create migration
```bash
alembic init alembic
alembic revision --autogenerate -m "Initial schema"
alembic upgrade head
```

## 9. Testing

### `backend/test_db.py`
```python
import asyncio
from db_service import DatabaseService

async def test_database():
    db_service = DatabaseService()
    
    # Create user
    user = await db_service.create_user("test@example.com", "hashed_password")
    print(f"Created user: {user.email}")
    
    # Create analysis
    analysis = await db_service.create_analysis(user.id, "123456789012", "us-east-1")
    print(f"Created analysis: {analysis.id}")
    
    # Get analyses
    analyses = await db_service.get_user_analyses(user.id)
    print(f"User analyses: {len(analyses)}")

asyncio.run(test_database())
```

---

**Prompt 3 Implementation Complete!** Ready for Prompt 4 (React Frontend).
