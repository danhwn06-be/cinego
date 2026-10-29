# 01. Domain Specification

> Status: **v0.1, ready for schema design.**
>
> Tags used in this document:
> - **[Verified]**: checked against a source listed in section 10.
> - **[Decision]**: chosen by the project advisor. The owner may override; record any change in `decisions.md`.
> - **[Assumption]**: not verified. Needs confirmation before it is relied on.
>
> Rule IDs (for example `BR-SEAT-2`) are stable. Reference them in code comments, tests and commits.

## 1. Goals and scope

CineGo lets a customer pick a movie, a cinema and a showtime, choose seats on a seat map, pay, and receive an e-ticket.

Working assumption: a multi-branch cinema chain in Vietnam, similar to the CGV model. **[Assumption]**

**MVP success criterion:** two customers can never both hold or buy the same seat for the same showtime, and seats that a customer abandons return to sale automatically. Everything else is secondary to this.

## 2. User roles

| Role | In MVP | Notes |
|---|---|---|
| Guest customer | Yes | Books without an account, using name, email and phone. |
| Member | No | Accounts, history, loyalty. The data model keeps a nullable `user_id` on bookings so this can be added without rework. |
| Cinema staff | No | Ticket check-in. The `checked_in` booking status exists but nothing sets it in MVP. |
| Admin | No UI | Movies, cinemas, screens and showtimes come from seed data or SQL. No admin screens in MVP. |

## 3. Booking flow

| # | Customer action | System behavior |
|---|---|---|
| 1 | Opens the site and picks a city. | Lists movies that have upcoming showtimes in that city ("now showing") and movies with a future release date ("coming soon"). |
| 2 | Opens a movie. | Shows poster, synopsis, duration, genres, cast, trailer, and the **age rating** with its warning (BR-AGE-1..3). |
| 3 | Picks a date, a cinema and a showtime. | Showtimes are grouped by cinema, each showing start time, format, language/subtitle and screen type. Showtimes that are closed for sale (BR-SHOW-3) are shown disabled. |
| 4 | Sees the seat map for the showtime. | Each seat shows one of: available, held by someone else, booked, or selected by me. Seat type is visible. |
| 5 | Selects seats, then presses **Continue**. | The client only highlights seats; it decides nothing. On Continue the server tries to **hold all selected seats at once** (BR-SEAT-3). On success a 10-minute countdown starts. |
| 6 | Chooses a ticket category per seat and optional food combos. | Server recalculates the total on every change (BR-PRICE-1). |
| 7 | Enters contact details and pays. | Mock payment (BR-PAY-1). On success the booking becomes confirmed. |
| 8 | Sees the confirmation. | Shows booking code and QR code. |
| 9 | Later, looks up the booking by code plus email or phone. | May cancel within the allowed window (BR-CANCEL-1). |

**Failure paths the product must handle:**

- At step 5 one or more seats were taken in the meantime: nothing is held, the customer sees which seats were lost and the refreshed map.
- The countdown reaches zero: seats are released, the customer is told, and is sent back to the seat map of the same showtime.
- Payment fails: the hold stays alive until it expires; the customer may retry or change method.
- The customer leaves the page or presses back: seats are released on a best-effort basis, and by expiry otherwise (BR-SEAT-5).
- The showtime closes for sale while the customer is in the flow: an existing valid hold may still complete payment (BR-SHOW-3).

## 4. Business rules

### 4.1 Seat state and holds

| ID | Rule |
|---|---|
| BR-SEAT-1 | A seat is tracked **per showtime**. For one seat in one showtime the state is exactly one of `available`, `held`, `booked`. **[Decision]** |
| BR-SEAT-2 | A seat is `held` only while an unexpired hold covers it. An expired hold counts as `available` immediately, without waiting for any cleanup job. **[Decision]** |
| BR-SEAT-3 | Holding is **all or nothing**. If any requested seat is not available, no seat is held. **[Decision]** |
| BR-SEAT-4 | A hold lasts **10 minutes** from creation and cannot be extended. The duration is configurable (`SEAT_HOLD_MINUTES`). The 10-minute figure is common across booking-system designs and one ticketing service; I could not confirm CGV's own figure. **[Decision]** |
| BR-SEAT-5 | A hold is released when: it expires, the customer abandons or goes back, or the customer starts a new hold for a different showtime. A session has at most one active hold. **[Decision]** |
| BR-SEAT-6 | At most **8 seats** per booking. **[Decision]** |
| BR-SEAT-7 | The server is the only authority on expiry. The client countdown is for display. When the two disagree the server wins. **[Decision]** |
| BR-SEAT-8 | Confirmation re-checks that the hold is still valid at the moment of confirmation. A payment that completes after expiry must not confirm the booking if another customer has since taken a seat. **[Decision]** |

### 4.2 Showtimes

| ID | Rule |
|---|---|
| BR-SHOW-1 | A showtime has: movie, screen, start time, **format** (2D, 3D, IMAX, 4DX, ScreenX), audio language and subtitle language. Formats exist in real listings, for example "2D Vietnam Sub". **[Verified]** |
| BR-SHOW-2 | Two showtimes on the same screen must not overlap. Leave a 15-minute cleaning gap between the end of one and the start of the next. **[Decision]** |
| BR-SHOW-3 | A showtime is open for sale until **10 minutes before** its start time. **[Decision]** |
| BR-SHOW-4 | End time is derived: start + movie duration + 15 minutes. Not entered by hand. **[Decision]** |

### 4.3 Pricing

All prices are integer **VND**. No decimals anywhere.

| ID | Rule |
|---|---|
| BR-PRICE-1 | The **server computes every price**. The client displays the total returned by the server and never calculates its own. **[Decision]** |
| BR-PRICE-2 | Price of one seat line = showtime base price for the ticket category + seat type surcharge. **[Decision]** |
| BR-PRICE-3 | Real prices vary by cinema cluster, day, seat type, format and customer type, so the base price is stored **on the showtime**, one value per ticket category. This avoids building a pricing engine in MVP. **[Decision, based on Verified variation]** |
| BR-PRICE-4 | Ticket categories in MVP: `adult` and `student` (U22). Student tickets require proof at entry; the confirmation screen says so. Child and senior categories are later. **[Decision; U22 and proof of ID Verified]** |
| BR-PRICE-5 | Seat types in MVP: `standard`, `vip`, `couple`. Surcharges are stored per seat type. **[Verified that all three exist at CGV]** |
| BR-PRICE-6 | A `couple` seat is **one sellable unit that seats two people**. Its price is 2 x base price + couple surcharge. How CGV actually prices a couple seat was unclear in the source. **[Assumption]** |
| BR-PRICE-7 | The final price of each seat line and food line is **frozen when payment is confirmed** and stored on the booking. Later price changes never alter existing bookings. **[Decision]** |

### 4.4 Food and drink

| ID | Rule |
|---|---|
| BR-FOOD-1 | Food and drink appear in the booking flow as a step after seat selection. **[Verified]** (CGV's app offers food and drink alongside seat choice.) |
| BR-FOOD-2 | MVP has a global catalog of combos with a fixed price. No stock tracking, no per-cinema menu. Food is collected at the cinema counter. **[Decision]** |
| BR-FOOD-3 | Buying food separately from a ticket is out of scope. (CGV added this feature in 2023.) **[Decision]** |

### 4.5 Payment

| ID | Rule |
|---|---|
| BR-PAY-1 | Payment is **simulated**. No real money, no real gateway. Methods offered: card, e-wallet, bank transfer, all fake. **[Decision]** |
| BR-PAY-2 | The mock gateway must be **deterministic** so tests can force each outcome: a documented test input that always succeeds, one that always fails, and one that is slow. **[Decision]** |
| BR-PAY-3 | Confirming a payment is **idempotent**. Repeating the same confirmation request returns the original result and never creates a second payment or booking. **[Decision]** |
| BR-PAY-4 | Promotions and payment-method discounts (for example e-wallet deals) are out of scope. **[Decision]** |

### 4.6 Booking and tickets

| ID | Rule |
|---|---|
| BR-BOOK-1 | A **booking** is one order for one showtime. It contains one **ticket** per seat and optional food lines. Tickets cannot be split across bookings. **[Decision]** |
| BR-BOOK-2 | Booking statuses: `pending` (hold active, unpaid), `confirmed`, `expired`, `cancelled`, `checked_in`. Transitions are in section 6. **[Decision]** |
| BR-BOOK-3 | Each confirmed booking has one **booking code**: 8 uppercase characters from an alphabet that excludes `0`, `O`, `1`, `I`, unique across the system. **[Decision]** |
| BR-BOOK-4 | The QR code encodes only the booking code. Validity is always checked on the server at scan time, never trusted from the QR content. **[Decision]** |
| BR-BOOK-5 | CGV customers collect printed tickets at a kiosk or counter after booking online. CineGo does not: the QR code on screen is the ticket. **[Verified for CGV, Decision for CineGo]** |

### 4.7 Cancellation and refunds

| ID | Rule |
|---|---|
| BR-CANCEL-1 | A confirmed booking may be cancelled by the customer **at least 60 minutes before** showtime start. CGV's customer-initiated refund uses the same 60-minute limit. **[Verified for CGV, Decision for CineGo]** |
| BR-CANCEL-2 | Cancellation applies to the **whole booking**, never to part of it. Food lines are refunded with it. **[Decision]** |
| BR-CANCEL-3 | A cancelled booking returns its seats to `available` immediately. **[Decision]** |
| BR-CANCEL-4 | The refund is simulated and goes back to the original payment method. **Deliberate simplification:** CGV refunds online payments as a gift card valid for one month, and limits each account to 2 refunds per month. CineGo has no gift cards and no accounts in MVP, so both are out of scope. **[Decision]** |
| BR-CANCEL-5 | If the cinema cancels a showtime, every confirmed booking for it is cancelled and fully refunded. There is no admin screen for this in MVP, but the rule is recorded so the data model supports it. **[Decision]** |

### 4.8 Age ratings

Vietnam's film classification is defined in the Law on Cinema 2022, Article 32, clause 1. **[Verified]**

| Rating | Meaning |
|---|---|
| P | All ages. |
| K | Under 13 only when accompanied by a parent or guardian. |
| T13 | 13 and over. |
| T16 | 16 and over. |
| T18 | 18 and over. |
| C | Not permitted for distribution. |

| ID | Rule |
|---|---|
| BR-AGE-1 | Every movie has exactly one rating from P, K, T13, T16, T18. **C films are never stored or shown.** **[Decision]** |
| BR-AGE-2 | The rating is displayed on the movie list, movie detail and seat-selection screens. **[Decision]** |
| BR-AGE-3 | For K, T13, T16 and T18 the customer must tick an acknowledgement before payment ("I confirm every viewer meets the age requirement" or, for K, "a parent or guardian will accompany viewers under 13"). Real age checking happens at the cinema entrance, outside the system. One source says cinemas face penalties for admitting under-age viewers. **[Decision; penalty claim unverified]** |

### 4.9 Guest identity

| ID | Rule |
|---|---|
| BR-ID-1 | A guest is identified during the flow by an opaque **session token** issued by the server. It scopes the guest's active hold. **[Decision]** |
| BR-ID-2 | At payment the guest supplies full name, email and phone. These are stored on the booking. **[Decision]** |
| BR-ID-3 | Looking up a booking later requires the **booking code plus the email or phone** used at purchase. The code alone is not enough. **[Decision]** |
| BR-ID-4 | Personal data is limited to what BR-ID-2 lists. No passwords, no card numbers stored (payment is simulated). **[Decision]** |

## 5. Entities

Business-level view. Column types and keys are decided in `02-database-schema.md`.

| Entity | Key attributes |
|---|---|
| City | name |
| Cinema | city, name, address |
| Screen | cinema, name, screen type (standard, IMAX, 4DX, ScreenX, Gold Class) |
| SeatType | code (`standard`, `vip`, `couple`), surcharge in VND, admissions (1 or 2) |
| Seat | screen, row label, column number, seat type, column span. This is the physical seat template of a screen. |
| Movie | title, original title, synopsis, duration in minutes, release date, age rating, poster URL, trailer URL, director |
| Genre | name (many-to-many with Movie) |
| Person (cast) | name (many-to-many with Movie, optional in MVP) |
| Showtime | movie, screen, start time, format, audio language, subtitle language, adult price, student price |
| Booking | showtime, status, hold expiry, booking code, contact name, email, phone, total, nullable user, created and confirmed times |
| Ticket | booking, seat, ticket category, price (frozen at confirmation) |
| FoodItem | name, description, price, active flag |
| BookingFood | booking, food item, quantity, price (frozen at confirmation) |
| Payment | booking, method, amount, status (`pending`, `succeeded`, `failed`, `refunded`), idempotency key |

## 6. Lifecycles

**Seat in a showtime** (BR-SEAT-1, 2, 5, 8)

| From | To | Trigger |
|---|---|---|
| available | held | Customer holds seats. |
| held | available | Hold expires, abandoned, or replaced. |
| held | booked | Booking confirmed with a valid hold. |
| booked | available | Booking cancelled. |

**Booking** (BR-BOOK-2)

| From | To | Trigger |
|---|---|---|
| (new) | pending | Seats held. |
| pending | confirmed | Payment succeeds while hold is valid. |
| pending | expired | Hold runs out or is abandoned. |
| confirmed | cancelled | Customer cancels within BR-CANCEL-1, or cinema cancels the showtime. |
| confirmed | checked_in | Staff scan (not built in MVP). |

`expired`, `cancelled` and `checked_in` are final.

## 7. Out of scope for MVP

Accounts and login, loyalty points, promotions and vouchers, gift cards, separate food purchase, staff check-in app, admin screens, partial cancellation, seat-gap rules (preventing a single orphan seat), waiting lists, notifications by email or SMS, multiple languages in the UI, real payment gateway.

## 8. Open questions

Each has a default so work is not blocked. Answer to override.

1. **Target market.** Default: a multi-branch cinema chain in Vietnam.
2. **Couple seat pricing** (BR-PRICE-6). Default: 2 x base + surcharge.
3. **Hold duration** (BR-SEAT-4). Default: 10 minutes. CGV's real figure is unknown.
4. **Sales cutoff** (BR-SHOW-3). Default: 10 minutes before start.
5. **Refund model** (BR-CANCEL-4). Default: simulated refund to the original method, no gift cards.
6. **Seat limit** (BR-SEAT-6). Default: 8.

## 9. Glossary (Vietnamese business term to English identifier)

Use the English identifier in code, schema and docs. The Vietnamese term is only for the project owner's reference.

| Vietnamese | English identifier |
|---|---|
| phim | movie |
| rạp | cinema |
| phòng chiếu | screen |
| suất chiếu | showtime |
| ghế | seat |
| ghế đôi (Sweetbox) | couple seat |
| giữ ghế | seat hold |
| vé | ticket (one seat line of a booking) |
| đơn đặt vé | booking |
| mã đặt vé | booking code |
| combo bắp nước | food combo |
| định dạng chiếu (2D, 3D, 4DX) | format |
| phân loại độ tuổi | age rating |
| hoàn vé | cancellation and refund |

## 10. Sources and confidence

Read directly:

- Age ratings: [Phim loại T18 chỉ được xem khi đủ bao nhiêu tuổi (Thư Viện Pháp Luật)](https://thuvienphapluat.vn/hoi-dap-phap-luat/phim-loai-t18-chi-duoc-xem-khi-du-bao-nhieu-tuoi-tro-len-138027930.html) and [Phân loại phim theo độ tuổi để bảo vệ trẻ em (Nhân Dân)](https://nhandan.vn/phan-loai-phim-theo-do-tuoi-de-bao-ve-tre-em-post709849.html). Both agree on the six categories.
- Price structure: [Giá vé CGV mới nhất tháng 10/2026 (ViettelStore, 13/10/2026)](https://viettelstore.vn/tin-tuc/gia-ve-cgv). The article contradicts itself in places (for example the Wednesday price and the Sweetbox surcharge), so treat figures as ranges.
- CGV refund feature: [CGV ra mắt tính năng hoàn vé (Kenh14, 12/05/2023)](https://kenh14.vn/cgv-ra-mat-tinh-nang-moi-cho-phep-nguoi-dung-hoan-ve-20230511200712033.chn). This is a 2023 article; CGV may have changed the policy since.
- Earlier research: [CGV ticket collection and booking guides](https://fptshop.com.vn/tin-tuc/thu-thuat/cach-lay-ve-xem-phim-khi-dat-online-cgv-198118), [iTicket FAQ (10-minute hold)](https://web-cp.iticket.az/en/faq), [Codemia booking system design](https://codemia.io/courses/ood_fundamentals/design_a_movie_ticket_booking_system).

Not readable: CGV's own FAQ, terms and announcement pages block automated access, so CGV's official hold time, refund terms and ticket rules were **not** verified at source.

**Reference price ranges for seed data** (VND, adult, from the ViettelStore article; indicative only):

| Item | Range |
|---|---|
| 2D weekday | 50,000 to 105,000 |
| 2D weekend and holiday | about +15,000 to +26,000 over weekday |
| 3D surcharge | about 32,000 weekday, about 53,000 weekend |
| VIP seat surcharge | about 5,000 to 5,500 |
| Sweetbox surcharge | about 16,000 to 30,000 |
| IMAX | 123,000 to 183,000 |
| Student (U22) | about 60,000 to 70,000 |