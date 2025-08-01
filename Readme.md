# RecycleBIM-ProductManager


 🛠️ Technology stack

| Layer            | Choice / version         | Notes                         |
| ---------------- | ------------------------ | ----------------------------- |
| Language         | Python 3.9               |                               |
| Framework        | Django 4.2 + DRF 3.15    | REST handling & admin         |
| DB               | MySQL 8.0                | spatial filters via `lat/lng` |
| Auth             | Bearer token + `Referer` | one token per **Marketplace** |
| Containerisation | Docker / docker-compose  | single-command start          |


💾 Data model

| Model                           | Purpose (one-liner)                                |
| ------------------------------- | -------------------------------------------------- |
| **Project**                     | A demolition project (operator, location, dates)   |
| **BuildingComponent**           | Individual IFC-derived component (wall, window, …) |
| **BuildingComponentCollection** | Logical group of components offered together       |
| **BuildingMaterial**            | Single material record (steel, concrete, …)        |
| **BuildingMaterialCollection**  | Aggregated bulk material lot                       |
| **Marketplace**                 | External client onboarding info & API token        |
| **Order** / **Offer**           | Buy / sell flows with nested line-items            |


🔌 **REST API**

**1 Authentication & headers**

Authorization: Bearer <marketplace - token>

Referer: <your - frontend  -origin>     # must start with the URL registered in the Marketplace entry

Accept: application/json

Content-Type: application/json      # for POST/PUT/PATCH

If either header is missing or the token doesn’t belong to the Marketplace that matches Referer, the API returns 401 Unauthorized.

**2 Quick-look endpoint matrix**
| Path                                    | Main verbs             | Purpose                        |
| --------------------------------------- | ---------------------- | ------------------------------ |
| `/projects/`                            | `GET`                  | List demolition projects       |
| `/projects/{id}/`                       | `GET`                  | Project detail                 |
| `/building-components/`                 | `GET`                  | List individual components     |
| `/building-components/{guid}/`          | `GET`                  | Component detail               |
| `/building-materials/grouped/`          | `GET`                  | Aggregated materials (grouped) |
| `/building-materials/`                  | `GET`                  | List individual materials      |
| `/building-materials/{guid_ind}/`       | `GET`                  | Material detail                |
| `/building-material-collections/`       | `GET`                  | Lots of materials              |
| `/building-material-collections/{id}/`  | `GET`                  | Material-collection detail     |
| `/building-component-collections/`      | `GET`                  | Bundles of components          |
| `/building-component-collections/{id}/` | `GET`                  | Component-collection detail    |
| `/orders/`                              | `GET POST`             | Buyer orders                   |
| `/orders/{id}/`                         | `GET PUT PATCH DELETE` | Order detail & update          |
| `/offers/`                              | `GET POST`             | Seller offers                  |
| `/offers/{id}/`                         | `GET PUT PATCH DELETE` | Offer detail & update          |
| `/marketplace/`                         | `GET POST`             | Marketplace registry           |
| `/marketplace/{id}/`                    | `GET PUT PATCH DELETE` | Marketplace detail & update    |

**3 Resource-by-resource details**

**3.1 Projects**
List: GET /projects/

Filters (all optional):

-demolition_operator_name (contains, text)

-type_of_building (one of Residential | Commercial | …)

-demolition_start_date_after (YYYY-MM-DD)

-demolition_start_date_before

-estimated_end_date_after / before

-lat, lng, distance (km – returns projects within radius)

Search: same fields as filters plus full-text on demolition_operator_name.

Detail: GET /projects/{id}/
<details> <summary>Sample response</summary>
{
  "id": 42,
  "demolition_operator_name": "ACME Demolition Ltd.",
  "operator_email": "contact@acme.com",
  "location_latitude": 40.123,
  "location_longitude": -8.567,
  "model_reference_number": "RB-2025-0001",
  "type_of_building": "Industrial",
  "year_of_construction": 1978,
  "available_documentation": "BIM IFC",
  "demolition_start_date": "2025-09-01",
  "estimated_end_date": "2026-02-15"
}
</details>

**3.2 Building components**
List: GET /building-components/

Filters: mp_category, description, available_from, available_until, fast_sale (bool), project-level filters (demolition_operator_name, date window, geodistance).

Detail: GET /building-components/{guid}/ (IFC GUID as path variable).

Response fields match the BuildingComponentSerializer (image reference, geometry file path, quantity, etc.).

**3.3 Building materials (individual & grouped)**
| Endpoint                          | Description                                       | Extra filters                                        |
| --------------------------------- | ------------------------------------------------- | ---------------------------------------------------- |
| `/building-materials/`            | Raw rows of materials                             | `mat_quantity`                                       |
| `/building-materials/{guid_ind}/` | One row                                           | –                                                    |
| `/building-materials/grouped/`    | Same rows **aggregated** by material & waste-code | `waste_code`, `declared_unit`, price/quantity ranges |


Grouped responses add total_material_quantity, price_per_declared_unit, and a list of contributing guid_inds.

**3.4 Collections (lots / bundles)**
Materials: GET /building-material-collections/

Components: GET /building-component-collections/

Shared filters: lat, lng, distance, available_from/available_until, plus free-text description.
Only collections with positive quantity (total_material_quantity > 0) or active status (status='A') are returned.

**3.5 Marketplace flow (orders & offers)**
Buyer creates an order

POST /orders/ – body uses the OrderSerializer.

Sellers answer with one or more offers

POST /offers/ with an order foreign-key and either offer_building_components[] or offer_building_materials[].

Any side can update or cancel via standard REST verbs on /orders/{id}/ or /offers/{id}/.

Marketplace records (/marketplace/) simply store name, API URL, and auth token.

**4 Pagination & errors**

Pagination uses DRF’s default page / page_size query parameters (page-number style).

Standard HTTP status codes:

200 – OK

201 – Created (POST)

400 – Validation error ({"field": ["msg", …]})

401 / 403 – Missing or invalid token / referer

404 – Not found (bad id or GUID)

