# Cexplorer Native Ranking – Calculation Formula

The native **Cexplorer ranking** is based on a combination of objective on-chain data and a **randomized component** that changes **frequently**. This helps ensure **fairness between stake pools**, especially for smaller or newer operators, by preventing large pools from permanently dominating the top ranks.

> ⚠️ **Important:** One part of the formula is **randomized during every run**. This means the ranking is not static, preventing gaming or exploitation of a fixed score formula. The `randomiser` value is recalculated every time.

---

## Calculation Formula (SQL Structure)

```sql
json_build_object(
    'live_stake', least(1.3, greatest(0.8, (mps.live_stake + 1) / (
        SELECT avg(epoch_stake)
        FROM d.pool_stat
        WHERE epoch_no = (SELECT max(epoch_no) FROM d.pool_stat WHERE blocks_estimated > 2)
    ))),

    'epoch_stake', least(1.3, greatest(0.8, (mps.active_stake + 1) / (
        SELECT avg(epoch_stake)
        FROM d.pool_stat
        WHERE epoch_no = (SELECT max(epoch_no) FROM d.pool_stat WHERE blocks_estimated > 2)
    ))),

    'pledged', least(2.0, greatest(0.5, (mpd.pledged + 1) / (
        SELECT avg(pledged)
        FROM d.pool_stat
        WHERE epoch_no = (SELECT max(epoch_no) FROM d.pool_stat WHERE blocks_estimated > 2)
    ))),

    'delegators', least(4, greatest(0.5, ps.delegator / 300)),

    'pct_member_one', least(greatest(1, ps.pct_member), 3),

    'margin', (6 / greatest(((pool_update->'active'->>'margin')::real) * 100, 4)) * 1,

    'fixed_cost', (340000000 / greatest(((pool_update->'active'->>'fixed_cost')::real), 300000000)) * 0.5,

    'leverage', (1 - least(1, greatest(least(mpd.pledged, mpd.pledged), mps.live_stake / 350) / mps.live_stake) * 4) + 3,

    'metadata', (CASE WHEN pool_name->>'ticker' IS NULL THEN 0.01 ELSE 1 END),
    'metadata_ext',(SELECT CASE WHEN EXISTS (SELECT 1 FROM d.image WHERE hash = a.pool_id) THEN 2 ELSE 1 END),

    'randomiser', (SELECT 0.8 + random() * 0.4)
) AS multiplier,

json_build_object(
    'basic', 100,
    'epochs', least(200, mps.epochs)
) AS points
