Written for: the QA testers who'll be testing the Rate Analysis feature in the Electrical (BD) Estimation screen.

---

# QA Handover: PWD Rate Analysis (Electrical BD Estimation)

**Where:** Project → Estimation → Electrical (BD) → the **Symbol Count** and **Cable Pricing** tabs, in the **Unit Rate** column.
**What it does:** Rows that have a PWD rate analysis show a small breakdown icon in the Unit Rate cell. Clicking it opens a popover with the PWD cost breakdown. Users can edit some of the values, see the rate recalculate live, and click **Apply Changes** to save. Saving updates the row's Unit Rate, Price and the sheet summary.

### Suggested test items
| Format | Item code | Notes |
|---|---|---|
| Single-group table | `2.1.2.1` (11 KV RMU), `6.20.11` (LED Tube Lamp) | `6.20.11` has a blank "Installation" amount |
| Multi-group table | `8.1.1.1.1`, `8.1.1.29` (Passenger Lifts) | Two rate groups: "Price up to 3 stop" and "Next per stop" |
| Detailed breakup (sections A/B/C + Summary) | `1.1.2.2` (Fan Point Wiring), `1.18.1.1.1`, `1.21.1` | `1.1.2.2` should show **1097** by default |
| No rate analysis | Any Subhead 13 item (Vehicle spare parts) | The icon should not appear |

---

## 1. When the icon appears
1. A row with a PWD item that has a rate analysis shows the icon. Hovering it shows the tooltip "View PWD rate analysis for <item>".
2. The icon does not appear on rows with no PWD item, on custom or unmatched rows, or on Subhead 13 items.
3. Turn on **page-wise** view. The icon shows on the parent rows but not on the page sub-rows.
4. Check both the **Symbol Count** and **Cable Pricing** tabs. The **Load Calculate** tab should not have the icon.
5. Clicking the icon opens the popover and does not open the inline Unit Rate editor. Double-clicking the icon should not start editing the cell either.
6. Double-clicking elsewhere in the Unit Rate cell should still open the inline editor as before (regression check).

## 2. Loading and error states
7. While the popover loads it shows a skeleton, then the breakdown.
8. Each time the popover opens it fetches fresh data. Close it and reopen it to confirm the latest saved values come back.
9. If the API returns 404 (for example, the row's item changed so there's no analysis), it shows "Rate analysis is unavailable for this row."
10. Other API failures show "Couldn't load the rate analysis." with a **Retry** button, and Retry works.
11. Open it while offline (DevTools → Offline). It shows "Waiting for a network connection…" and recovers once you're back online.

## 3. How the breakdown looks
12. The header shows the item code, description, **Unit** and **Base** quantity.
13. **Single-group table:** no rate-group switcher, or one disabled "Standard Rate" chip. The heading reads "Cost component breakdown (Direct tabular analysis)".
14. **Multi-group (Lifts):** both group buttons show, and the active group is highlighted.
15. **Detailed breakup:** sections A/B/C are shown as collapsible blocks with a title and section total. They are open by default and can be collapsed and expanded. The heading reads "(Summary)".
16. Table columns are Ref ID, Component Label, Formula / Source Refs, Factor % and Amount. Missing values show "—".
17. Amounts use the `৳` symbol with up to 2 decimals. Factor % shows 2 to 4 decimals with a `%` sign.
18. The footer row "Math Verified Final Rate · <unit>" shows the rounded whole-number rate.
19. Layout: the popover is about 760px wide, fits on smaller screens, and scrolls inside itself when the content is tall. Check that it doesn't overflow the page.
20. Dark mode renders correctly.

## 4. Editing and live recalculation
21. Only fields the backend marks as editable show an input box:
    - Section lines: Qty, Rate, Factor % and/or Amount
    - Cost components: Factor % (percentage and VAT rows) and Amount (direct-input rows)
    - Everything else is read-only text.
22. After you change a value, "Recalculating…" appears and the totals, section totals and final rate update after about 0.3 seconds.
23. **VAT check on `1.1.2.2`:** change the VAT Factor % from `8.11` to `11.11`. The final rate should go from **1097 to 1127**.
24. Type quickly across several fields. Only the final values should be calculated, with no flicker back to older results.
25. Change a value and then type the original value back. **Apply Changes** should become disabled again.
26. `10` and `10.0` count as the same value, so changing one to the other shouldn't recalculate.

### Input validation
27. These should be accepted: `5`, `0`, `0.5`, `.5`, `1e3`.
28. These should be rejected: `-5`, `5.`, `1,000`, `abc`, `5%`, spaces. The field is marked invalid, the message "Enter a non-negative number or clear the field." shows, and **Apply Changes** is disabled.
29. Clear a field completely. This counts as "no value" and should recalculate. For a cleared Factor %, the component should stop applying a factor.
30. Fix an invalid field. The error message goes away and recalculation resumes.

### Calculation errors
31. Make `/rate-analysis/calculate` fail (block it in DevTools or go offline). The message "Couldn't recalculate…" shows with a **Retry calculation** link, and the link works.
32. If the backend returns a validation message, that message is shown instead of the generic one.

## 5. Rate groups (multi-group items)
33. Switch the group without editing anything. The breakdown recalculates and **Apply Changes** becomes enabled.
34. Apply after switching groups. The row's Unit Rate should equal the new group's final rate.
35. Edit a value in group 1, switch to group 2, then apply. Confirm whether the group 1 edit was saved too by reopening and switching back. **Please report what actually happens.** The current code keeps edits from both groups.

## 6. Saving (Apply Changes)
36. **Apply Changes** is disabled until there's a valid change that has finished recalculating.
37. While saving, the button reads "Saving…", every input and Cancel are disabled, and the popover can't be closed by clicking outside or pressing Esc.
38. On success, a toast says "Rate analysis saved.", the popover closes, and without a page refresh:
    - the row's Unit Rate equals the new final rate
    - the row's Price equals Qty × the new rate
    - the sheet summary/totals update
39. After a save, the icon looks "edited" (light highlighted background) and the tooltip ends in "(edited)".
40. Reopen the popover. It shows the saved custom values, not the official baseline.
41. Refresh the page. The edits, the new rate and the "edited" state are all still there.
42. Make a bulk save or other edits elsewhere in the sheet. The saved rate analysis should not be lost.
43. If the save fails (go offline or block the PATCH), an inline error ends with "Retry with Apply Changes.", the popover stays open, and your edits are kept. Retrying succeeds.
44. Double-click **Apply Changes** quickly. Only one save request should be sent.

## 7. Cancel, close and discarding edits
45. **Cancel**, Esc and clicking outside all close the popover and throw away unsaved edits, with no confirmation prompt. Reopen and confirm the saved values are shown.
46. Keyboard: Esc closes the popover. Pressing keys inside the inputs should not trigger the table's keyboard shortcuts (arrows, Enter, Delete).

## 8. Interactions with other features
47. Change a row's PWD item (search modal or editing the item). Its custom breakdown resets: the "edited" state clears, and reopening shows the new item's official breakdown.
48. Run **Quick Optimize** on both the Symbol Count and Cable Pricing tabs. Rows it overwrites lose their custom breakdown. Also check that Quick Optimize on the Cable Pricing tab now runs the cable optimisation, since it was rewired in this branch.
49. Edit the Unit Rate inline after saving a rate analysis. Check that the row updates and note whether the "edited" state changes, and report it.
50. **Export** (Excel/PDF/CSV): exported rates and prices match the rate analysis you saved.

## 9. Accessibility (quick pass)
51. The icon, every input and the group buttons can be reached and used by keyboard, and focus outlines are visible.
52. A screen reader announces the input labels (e.g. "Factor % for Contractor's Profit"), the active group, and the status messages ("Recalculating…" and errors).

---

## Not in this release
These exist in the backend API but have **no UI yet**, so QA shouldn't log them as bugs:
- **Reset to official rate** (restoring the PWD default)
- **Adding or deleting lines** inside a section
- Editing the **Base qty**
- Previewing the rate analysis from the PWD search modal

**Backend tests** for reference: `cd backend && pytest --noconftest tests/test_pwd_rate_analysis.py tests/test_pwd_estimation_api.py -v`

---

I didn't change any code; the list comes from reading the commits and source files. Items 35 and 49 are the ones where I'm least sure what the intended behaviour is, so it's worth deciding the expected result before QA starts. I can also put this in a shared doc for the team.
