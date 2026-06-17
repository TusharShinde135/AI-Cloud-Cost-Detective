# AWS Backend Implementation Guide - Prompt 2

Complete code examples and setup instructions for AI-powered cost analysis using OpenAI API.

## 1. Dependencies

Add to `requirements.txt`:
```
openai>=1.3.0
python-dotenv
pydantic
```

Install:
```bash
pip install openai python-dotenv pydantic
```

## 2. AI Analyzer Module

### `backend/ai_analyzer.py`
```python
import openai
import os
import json
from typing import Dict, List
from dotenv import load_dotenv

load_dotenv()

class AIAnalyzer:
    def __init__(self):
        self.openai_api_key = os.getenv("OPENAI_API_KEY")
        if not self.openai_api_key:
            raise ValueError("OPENAI_API_KEY not set in environment variables")
        openai.api_key = self.openai_api_key
    
    async def analyze(self, resources: Dict) -> Dict:
        """
        Analyze AWS resources for cost optimization opportunities
        
        Args:
            resources: Dictionary containing scanned AWS resources
        
        Returns:
            Dictionary with analysis results, recommendations, and estimated savings
        """
        try:
            # Build the analysis prompt
            prompt = self._build_prompt(resources)
            
            # Call OpenAI API
            response = openai.ChatCompletion.create(
                model="gpt-4",
                messages=[
                    {
                        "role": "system",
                        "content": "You are an expert AWS cost optimization consultant. Analyze AWS resources and provide specific, actionable recommendations to reduce cloud costs."
                    },
                    {
                        "role": "user",
                        "content": prompt
                    }
                ],
                temperature=0.7,
                max_tokens=2000
            )
            
            # Parse the response
            analysis_text = response.choices[0].message.content
            analysis = self._parse_analysis(analysis_text, resources)
            
            return analysis
        
        except openai.error.AuthenticationError:
            raise Exception("Invalid OpenAI API key")
        except openai.error.RateLimitError:
            raise Exception("OpenAI API rate limit exceeded")
        except Exception as e:
            raise Exception(f"OpenAI analysis failed: {str(e)}")
    
    def _build_prompt(self, resources: Dict) -> str:
        """Build the analysis prompt from scanned resources"""
        
        ec2_summary = self._summarize_ec2(resources.get('ec2_instances', []))
        rds_summary = self._summarize_rds(resources.get('rds_databases', []))
        s3_summary = self._summarize_s3(resources.get('s3_buckets', []))
        lambda_summary = self._summarize_lambda(resources.get('lambda_functions', []))
        ebs_summary = self._summarize_ebs(resources.get('ebs_volumes', []))
        
        prompt = f"""
Analyze the following AWS resources for cost optimization opportunities:

### EC2 INSTANCES
{ec2_summary}

### RDS DATABASES
{rds_summary}

### S3 BUCKETS
{s3_summary}

### LAMBDA FUNCTIONS
{lambda_summary}

### EBS VOLUMES
{ebs_summary}

Please provide:
1. Executive Summary (2-3 sentences)
2. High-Priority Issues (severity: HIGH) with estimated monthly savings
3. Medium-Priority Issues (severity: MEDIUM)
4. Low-Priority Issues (severity: LOW)
5. Quick Wins (top 3 actions for immediate cost reduction)
6. AWS CLI or Terraform commands to implement fixes where applicable
7. Total estimated monthly savings

Format each recommendation as:
- **Issue**: [Description]
- **Severity**: [HIGH/MEDIUM/LOW]
- **Current State**: [What's happening]
- **Recommended Action**: [What to do]
- **Estimated Savings**: $XXX/month
- **Fix Command**: [AWS CLI command]

Be specific with instance types, sizes, and actual recommendations.
"""
        return prompt
    
    def _summarize_ec2(self, instances: List[Dict]) -> str:
        """Summarize EC2 instances"""
        if not instances:
            return "No EC2 instances found."
        
        summary = f"Total instances: {len(instances)}\n"
        
        # Count by instance type
        instance_types = {}
        running_count = 0
        
        for instance in instances:
            inst_type = instance.get('type', 'unknown')
            instance_types[inst_type] = instance_types.get(inst_type, 0) + 1
            
            if instance.get('state') == 'running':
                running_count += 1
        
        summary += f"Running instances: {running_count}\n"
        summary += "Instance types breakdown:\n"
        for inst_type, count in sorted(instance_types.items()):
            summary += f"  - {inst_type}: {count}\n"
        
        return summary
    
    def _summarize_rds(self, databases: List[Dict]) -> str:
        """Summarize RDS databases"""
        if not databases:
            return "No RDS databases found."
        
        summary = f"Total databases: {len(databases)}\n"
        
        engines = {}
        for db in databases:
            engine = db.get('engine', 'unknown')
            engines[engine] = engines.get(engine, 0) + 1
            summary += f"  - {db['id']} ({db['instance_class']}) {db['storage_gb']}GB [{db['engine']}]\n"
        
        return summary
    
    def _summarize_s3(self, buckets: List[Dict]) -> str:
        """Summarize S3 buckets"""
        if not buckets:
            return "No S3 buckets found."
        
        return f"Total buckets: {len(buckets)}\n" + "\n".join([f"  - {b['name']}" for b in buckets[:10]])
    
    def _summarize_lambda(self, functions: List[Dict]) -> str:
        """Summarize Lambda functions"""
        if not functions:
            return "No Lambda functions found."
        
        summary = f"Total functions: {len(functions)}\n"
        
        total_memory = sum(f.get('memory_mb', 128) for f in functions)
        avg_memory = total_memory // len(functions) if functions else 0
        
        summary += f"Total memory allocated: {total_memory}MB\n"
        summary += f"Average memory per function: {avg_memory}MB\n"
        
        return summary
    
    def _summarize_ebs(self, volumes: List[Dict]) -> str:
        """Summarize EBS volumes"""
        if not volumes:
            return "No EBS volumes found."
        
        summary = f"Total volumes: {len(volumes)}\n"
        
        total_storage = sum(v.get('size_gb', 0) for v in volumes)
        summary += f"Total storage: {total_storage}GB\n"
        
        # Count by type
        volume_types = {}
        for vol in volumes:
            vol_type = vol.get('type', 'unknown')
            volume_types[vol_type] = volume_types.get(vol_type, 0) + 1
        
        summary += "Volume types:\n"
        for vol_type, count in sorted(volume_types.items()):
            summary += f"  - {vol_type}: {count}\n"
        
        return summary
    
    def _parse_analysis(self, analysis_text: str, resources: Dict) -> Dict:
        """
        Parse OpenAI response into structured format
        """
        issues = self._extract_issues(analysis_text)
        total_savings = self._extract_total_savings(analysis_text)
        
        return {
            "summary": self._extract_summary(analysis_text),
            "issues": issues,
            "high_severity": [i for i in issues if i.get('severity') == 'HIGH'],
            "medium_severity": [i for i in issues if i.get('severity') == 'MEDIUM'],
            "low_severity": [i for i in issues if i.get('severity') == 'LOW'],
            "quick_wins": self._extract_quick_wins(analysis_text),
            "estimated_monthly_savings": total_savings,
            "estimated_annual_savings": total_savings * 12 if total_savings else 0,
            "raw_analysis": analysis_text
        }
    
    def _extract_summary(self, text: str) -> str:
        """Extract executive summary"""
        lines = text.split('\n')
        summary_lines = []
        
        for i, line in enumerate(lines):
            if 'executive summary' in line.lower() or 'summary' in line.lower():
                # Get next 2-3 lines after summary header
                for j in range(i+1, min(i+4, len(lines))):
                    if lines[j].strip():
                        summary_lines.append(lines[j].strip())
                break
        
        return ' '.join(summary_lines) if summary_lines else "Analysis complete"
    
    def _extract_issues(self, text: str) -> List[Dict]:
        """Extract individual issues from analysis"""
        issues = []
        
        # Simple parsing - look for issue patterns
        lines = text.split('\n')
        current_issue = {}
        
        for line in lines:
            line = line.strip()
            
            if line.startswith('**Issue**:'):
                if current_issue:
                    issues.append(current_issue)
                current_issue = {'title': line.replace('**Issue**:', '').strip()}
            
            elif line.startswith('**Severity**:'):
                severity = line.replace('**Severity**:', '').strip()
                current_issue['severity'] = self._parse_severity(severity)
            
            elif line.startswith('**Estimated Savings**:'):
                savings = line.replace('**Estimated Savings**:', '').strip()
                current_issue['estimated_savings'] = self._parse_savings(savings)
            
            elif line.startswith('**Fix Command**:'):
                cmd = line.replace('**Fix Command**:', '').strip()
                current_issue['fix_command'] = cmd
            
            elif line.startswith('**Recommended Action**:'):
                action = line.replace('**Recommended Action**:', '').strip()
                current_issue['recommendation'] = action
        
        if current_issue:
            issues.append(current_issue)
        
        return issues
    
    def _extract_quick_wins(self, text: str) -> List[str]:
        """Extract quick wins section"""
        quick_wins = []
        lines = text.split('\n')
        
        for i, line in enumerate(lines):
            if 'quick win' in line.lower():
                # Get next 5 lines
                for j in range(i+1, min(i+6, len(lines))):
                    if lines[j].strip() and not lines[j].startswith('#'):
                        quick_wins.append(lines[j].strip().lstrip('- '))
                break
        
        return quick_wins
    
    def _extract_total_savings(self, text: str) -> float:
        """Extract total estimated savings"""
        import re
        
        # Look for patterns like "$XXX/month" or "$XXX per month"
        matches = re.findall(r'\$[\d,]+(?:\.\d{2})?', text)
        
        if matches:
            # Get the largest number (likely the total)
            amounts = [float(m.replace('$', '').replace(',', '')) for m in matches]
            return max(amounts) if amounts else 0
        
        return 0
    
    def _parse_severity(self, severity_str: str) -> str:
        """Parse severity level"""
        severity_str = severity_str.upper()
        if 'HIGH' in severity_str:
            return 'HIGH'
        elif 'MEDIUM' in severity_str:
            return 'MEDIUM'
        else:
            return 'LOW'
    
    def _parse_savings(self, savings_str: str) -> float:
        """Parse savings amount"""
        import re
        
        # Extract number from string like "$500/month"
        match = re.search(r'\$?([\d,]+(?:\.\d{2})?)', savings_str)
        if match:
            return float(match.group(1).replace(',', ''))
        return 0
```

## 3. Update Main Backend

### Update `backend/main.py`
```python
# Add to imports
from ai_analyzer import AIAnalyzer
from aws_scanner import AWSScanner

# In @app.post("/api/analyze") update:
@app.post("/api/analyze")
async def analyze(request: dict):
    """
    Analyze AWS resources for cost optimization
    Expected payload: { "account_id": "123456789012", "region": "us-east-1" }
    """
    try:
        account_id = request.get("account_id")
        region = request.get("region")
        
        if not account_id or not region:
            raise HTTPException(status_code=400, detail="account_id and region required")
        
        # Scan resources
        resources = await aws_scanner.scan_resources(account_id, region)
        
        # Analyze with AI
        analysis = await ai_analyzer.analyze(resources)
        
        return {
            "account_id": account_id,
            "region": region,
            "resources_scanned": len(resources),
            "analysis": analysis,
            "timestamp": datetime.now().isoformat()
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## 4. OpenAI API Key Setup

### Get OpenAI API Key
1. Go to https://platform.openai.com/account/api-keys
2. Create new secret key
3. Add to `.env`:
```env
OPENAI_API_KEY=sk-...your-key...
```

## 5. Cost Analysis Examples

### Example Output Structure
```json
{
  "account_id": "123456789012",
  "region": "us-east-1",
  "resources_scanned": 45,
  "analysis": {
    "summary": "Your AWS account is over-provisioned with multiple opportunities for cost reduction...",
    "issues": [
      {
        "title": "Over-provisioned EC2 instances",
        "severity": "HIGH",
        "estimated_savings": 500,
        "recommendation": "Right-size t3.xlarge to t3.large (70% CPU usage average)",
        "fix_command": "aws ec2 modify-instance-attribute --instance-id i-123abc --instance-type t3.large"
      }
    ],
    "high_severity": [...],
    "medium_severity": [...],
    "low_severity": [...],
    "quick_wins": [
      "Terminate 3 unused EBS volumes ($45/month savings)",
      "Disassociate 2 unattached Elastic IPs ($14/month savings)",
      "Enable S3 lifecycle policies ($120/month savings)"
    ],
    "estimated_monthly_savings": 679.50,
    "estimated_annual_savings": 8154.00
  }
}
```

## 6. Testing

### `backend/test_analyzer.py`
```python
import asyncio
from ai_analyzer import AIAnalyzer

async def test_analyzer():
    analyzer = AIAnalyzer()
    
    # Sample resources
    mock_resources = {
        "ec2_instances": [
            {
                "id": "i-1234567890abcdef0",
                "type": "t3.xlarge",
                "state": "running",
                "launch_time": "2024-01-15T10:00:00Z",
                "tags": {"Name": "web-server", "Environment": "prod"}
            }
        ],
        "rds_databases": [
            {
                "id": "mydb",
                "engine": "postgres",
                "instance_class": "db.r5.2xlarge",
                "storage_gb": 500,
                "status": "available"
            }
        ],
        "s3_buckets": [
            {"name": "my-bucket-1", "created": "2023-01-01T00:00:00Z"}
        ],
        "lambda_functions": [
            {
                "name": "my-function",
                "runtime": "python3.11",
                "memory_mb": 1024,
                "timeout_sec": 300
            }
        ],
        "ebs_volumes": [
            {
                "id": "vol-123456",
                "size_gb": 100,
                "type": "gp2",
                "state": "available",
                "iops": 300
            }
        ],
        "load_balancers": [],
        "cloudwatch_metrics": []
    }
    
    # Run analysis
    result = await analyzer.analyze(mock_resources)
    
    print("Analysis Results:")
    print(f"Summary: {result['summary']}")
    print(f"High Severity Issues: {len(result['high_severity'])}")
    print(f"Estimated Monthly Savings: ${result['estimated_monthly_savings']}")
    print(f"Estimated Annual Savings: ${result['estimated_annual_savings']}")

asyncio.run(test_analyzer())
```

Run test:
```bash
python test_analyzer.py
```

## 7. Error Handling

```python
# Handle API errors gracefully
try:
    analysis = await ai_analyzer.analyze(resources)
except ValueError as e:
    # Missing API key
    return {"error": "OpenAI API key not configured", "status": 500}
except Exception as e:
    # OpenAI API errors
    return {"error": f"Analysis failed: {str(e)}", "status": 500}
```

## 8. Advanced Options

### Using GPT-4 Turbo (Faster & Cheaper)
```python
response = openai.ChatCompletion.create(
    model="gpt-4-turbo-preview",  # Faster
    messages=[...],
    temperature=0.7
)
```

### Using GPT-3.5 Turbo (Cheapest)
```python
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",  # Most economical
    messages=[...],
    temperature=0.7
)
```

---

**Prompt 2 Implementation Complete!** Ready for Prompt 3 (Database & WebSocket).
