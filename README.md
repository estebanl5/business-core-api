# 📄 PRD – CRM + HR API  

## 🎯 Objective
Design an **enterprise example API** that combines **CRM (Customer Relationship Management)** and **HR (Human Resources)** modules. The API must comply with enterprise-level **non-functional requirements (NFRs)**: security, compatibility, performance, and usability.  

---

## 🔒 Non-Functional Requirements (NFRs)

### Security
- Authentication via **Azure AD (OAuth2/OpenID Connect)**.  
- Authorization based on **roles and claims**:  
  - `crm.read`, `crm.write`, `crm.admin`  
  - `hr.read`, `hr.write`, `hr.admin`  
- HTTPS (TLS 1.2+) enforced.  
- Rate limiting and throttling with **Azure API Management**.  
- Audit logs for sensitive operations (e.g., salary updates, customer deletion).  

### Compatibility
- REST with **JSON** (OData optional).  
- **OpenAPI 3.0 (Swagger)** definition.  
- Versioning with prefix `/api/v1/`.  
- Webhooks for events (`customer.created`, `employee.created`).  

### Performance
- **Read requests (GET)** < 200 ms under normal load.  
- **Write requests (POST/PUT/PATCH)** < 500 ms.  
- Horizontal autoscaling (Azure App Service or AKS).  
- Caching for frequent queries (e.g., `GET /employees`).  

### Usability
- Developer portal with documentation (API Management).  
- Example requests/responses, SDK auto-generation.  
- Sandbox environment with dummy data.  
- Observability with **Application Insights** and Azure Monitor alerts.  

---

## 🔗 Endpoints

### CRM Module
**Customers**  
- `GET /api/v1/customers` → list customers *(role: `crm.read`)*  
- `GET /api/v1/customers/{id}` → get a customer *(role: `crm.read`)*  
- `POST /api/v1/customers` → create a customer *(role: `crm.write`)*  
- `PUT /api/v1/customers/{id}` → replace customer data *(role: `crm.write`)*  
- `PATCH /api/v1/customers/{id}` → update customer partially *(role: `crm.write`)*  
- `DELETE /api/v1/customers/{id}` → delete customer *(role: `crm.admin`)*  

**Opportunities**  
- `GET /api/v1/opportunities` → list opportunities *(role: `crm.read`)*  
- `POST /api/v1/opportunities` → create opportunity *(role: `crm.write`)*  
- `PATCH /api/v1/opportunities/{id}` → update opportunity *(role: `crm.write`)*  
- `DELETE /api/v1/opportunities/{id}` → delete opportunity *(role: `crm.admin`)*  

---

### HR Module
**Employees**  
- `GET /api/v1/employees` → list employees *(role: `hr.read`)*  
- `GET /api/v1/employees/{id}` → get employee *(role: `hr.read`)*  
- `POST /api/v1/employees` → create employee *(role: `hr.write`)*  
- `PUT /api/v1/employees/{id}` → replace employee *(role: `hr.write`)*  
- `PATCH /api/v1/employees/{id}` → update employee partially *(role: `hr.write`)*  
- `DELETE /api/v1/employees/{id}` → delete employee *(role: `hr.admin`)*  

**Leaves / Absences**  
- `GET /api/v1/employees/{id}/leaves` → list leaves *(role: `hr.read`)*  
- `POST /api/v1/employees/{id}/leaves` → register leave *(role: `hr.write`)*  
- `PATCH /api/v1/leaves/{id}` → update leave *(role: `hr.write`)*  
- `DELETE /api/v1/leaves/{id}` → delete leave *(role: `hr.admin`)*  

---

## 🛡️ Endpoint Security Matrix

| Endpoint                  | Method | Minimum Role Required |
|---------------------------|--------|------------------------|
| `/customers`              | GET    | `crm.read`            |
| `/customers`              | POST   | `crm.write`           |
| `/customers/{id}`         | PUT    | `crm.write`           |
| `/customers/{id}`         | PATCH  | `crm.write`           |
| `/customers/{id}`         | DELETE | `crm.admin`           |
| `/employees`              | GET    | `hr.read`             |
| `/employees`              | POST   | `hr.write`            |
| `/employees/{id}`         | PUT    | `hr.write`            |
| `/employees/{id}`         | PATCH  | `hr.write`            |
| `/employees/{id}`         | DELETE | `hr.admin`            |

---

## 🧩 Prompt Usage
This PRD can be used with LLMs to:  
- Generate **OpenAPI 3.0 specification (Swagger YAML/JSON)**.  
- Create backend code stubs (Node.js, .NET, Python).  
- Define test cases (Postman collections, unit tests).  
- Draft CI/CD pipelines for Azure.  
