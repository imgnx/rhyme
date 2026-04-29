#!/usr/bin/env node

/**
 * rhyme
 *
 * Tiny CMUdict rhyme finder.
 *
 * Usage:
 *   rhyme wow
 *   rhyme wow --phones
 *   rhyme wow --key
 *   rhyme wow --limit 50
 *   rhyme wow --say
 *
 * Requires:
 *   python -m pip install cmudict
 *
 * Or if you're using npm and a different CMUdict package later,
 * we can swap the loader out.
 */

async function main() {
  const args = process.argv.slice(2)

  if (args.length === 0 || args.includes("--help") || args.includes("-h")) {
    printHelp()
    process.exit(0)
  }

  const word = args[0].toLowerCase()

  const showPhones = args.includes("--phones")
  const showKey = args.includes("--key")
  const shouldSay = args.includes("--say")

  const limitIndex = args.findIndex((arg) => arg === "--limit")
  const limit =
    limitIndex !== -1 && args[limitIndex + 1]
      ? Number.parseInt(args[limitIndex + 1], 10)
      : null

  const cmudictModule = await import("cmudict")

  const cmudictDefault = cmudictModule.default ?? cmudictModule
  const CMUDict = cmudictDefault.CMUDict ?? cmudictModule.CMUDict

  if (!CMUDict) {
    console.error("Could not find CMUDict export from cmudict module.")
    console.error("Module keys:", Object.keys(cmudictModule))
    process.exit(1)
  }

  const dict = new CMUDict()

  const phones = dict.get(word)

  if (!phones) {
    console.error(`No pronunciation found for: ${word}`)
    process.exit(1)
  }

  const key = rhymeKey(phones)

  if (showPhones) {
    console.log(`${word}\t${phones}`)
  }

  if (showKey) {
    console.log(`${word}\t${key}`)
  }

  const words = getKnownWords(dict)

  if (!words.length) {
    console.error("Could not inspect word list from this CMUDict package.")
    console.error("This package exposes dict.get(word), but not an obvious word list.")
    console.error("Use the single-word helpers for now:")
    console.error(`  rhyme ${word} --phones`)
    console.error(`  rhyme ${word} --key`)
    process.exit(1)
  }

  const rhymes = []

  for (const candidate of words) {
    const candidateWord = candidate.toLowerCase()

    if (candidateWord === word) continue

    const candidatePhones = dict.get(candidateWord)

    if (!candidatePhones) continue

    if (rhymeKey(candidatePhones) === key) {
      rhymes.push(candidateWord)
    }
  }

  const uniqueRhymes = [...new Set(rhymes)].sort()

  const output =
    Number.isFinite(limit) && limit > 0
      ? uniqueRhymes.slice(0, limit)
      : uniqueRhymes

  if (shouldSay) {
    await say(output.join(" "))
  } else {
    for (const rhyme of output) {
      console.log(rhyme)
    }
  }
}

function rhymeKey(phones) {
  const parts = phones.trim().split(/\s+/)

  for (let i = parts.length - 1; i >= 0; i--) {
    if (/\d$/.test(parts[i])) {
      return parts.slice(i).join(" ")
    }
  }

  return phones
}

/**
 * This tries to discover where the package keeps its internal dictionary.
 * Your current package exposes CMUDict#get(), but not much else publicly.
 * So this function pokes gently at common object shapes.
 */
function getKnownWords(dict) {
  const candidates = [
    dict.dict,
    dict.dictionary,
    dict.entries,
    dict.words,
    dict.data,
    dict._dict,
    dict._dictionary,
    dict._entries,
    dict._words,
    dict._data,
  ]

  for (const candidate of candidates) {
    if (!candidate) continue

    if (Array.isArray(candidate)) {
      return candidate
        .map((entry) => {
          if (typeof entry === "string") return entry
          if (Array.isArray(entry)) return entry[0]
          if (entry && typeof entry === "object") {
            return entry.word ?? entry[0]
          }
          return null
        })
        .filter(Boolean)
    }

    if (candidate instanceof Map) {
      return [...candidate.keys()]
    }

    if (typeof candidate === "object") {
      return Object.keys(candidate)
    }
  }

  /**
   * Last-resort object inspection.
   * Sometimes packages hide the useful object under one property name.
   */
  for (const key of Object.keys(dict)) {
    const value = dict[key]

    if (!value) continue

    if (value instanceof Map) {
      return [...value.keys()]
    }

    if (Array.isArray(value)) {
      return value
        .map((entry) => {
          if (typeof entry === "string") return entry
          if (Array.isArray(entry)) return entry[0]
          if (entry && typeof entry === "object") {
            return entry.word ?? entry[0]
          }
          return null
        })
        .filter(Boolean)
    }

    if (typeof value === "object") {
      const keys = Object.keys(value)

      if (keys.length > 1000) {
        return keys
      }
    }
  }

  return []
}

function say(text) {
  return new Promise((resolve, reject) => {
    const { spawn } = require("node:child_process")

    const child = spawn("say", [text], {
      stdio: "inherit",
    })

    child.on("error", reject)

    child.on("close", (code) => {
      if (code === 0) {
        resolve()
      } else {
        reject(new Error(`say exited with code ${code}`))
      }
    })
  })
}

function printHelp() {
  console.log(`rhyme

Usage:
  rhyme <word>
  rhyme <word> --phones
  rhyme <word> --key
  rhyme <word> --limit <number>
  rhyme <word> --say

Examples:
  rhyme wow
  rhyme wow --phones
  rhyme wow --key
  rhyme wow --limit 40
  rhyme wow --say
`)
}

main().catch((error) => {
  console.error(error.message)
  process.exit(1)
})