import React, { useState, useEffect } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";

const allTors = Array.from({ length: 23 }, (_, i) => 202 + i); // Includes 215

const getNextState = (current) => {
  switch (current) {
    case "Frei":
      return "Leerer Trailer benötigt";
    case "Leerer Trailer benötigt":
      return "Beladung";
    case "Beladung":
      return "Fertig";
    default:
      return "Frei";
  }
};

const getStyle = (status, disabled) => {
  if (disabled) return "bg-gray-200 text-gray-400 cursor-not-allowed";
  switch (status) {
    case "Frei":
      return "bg-white text-black border border-gray-400";
    case "Beladung":
      return "bg-yellow-300 text-black border border-yellow-500";
    case "Fertig":
      return "bg-green-600 text-white border border-green-700";
    case "Leerer Trailer benötigt":
      return "bg-red-600 text-white border border-red-700";
    case "Anders belegt":
      return "bg-purple-300 text-black border border-purple-500";
    default:
      return "";
  }
};

const getBackgroundColor = (status) => {
  switch (status) {
    case "Frei":
      return "bg-white";
    case "Beladung":
      return "bg-yellow-100";
    case "Fertig":
      return "bg-green-100";
    case "Leerer Trailer benötigt":
      return "bg-red-100";
    case "Anders belegt":
      return "bg-purple-100";
    default:
      return "";
  }
};

const getNotificationColor = (status) => {
  switch (status) {
    case "Frei":
      return "bg-white text-black";
    case "Beladung":
      return "bg-yellow-300 text-black";
    case "Fertig":
      return "bg-green-600 text-white";
    case "Leerer Trailer benötigt":
      return "bg-red-600 text-white";
    default:
      return "bg-gray-600 text-white";
  }
};

const removeOldArchiveEntries = (archive) => {
  const now = new Date();
  return archive.filter((entry) => {
    const entryTime = new Date(entry.timestamp);
    return now - entryTime < 12 * 60 * 60 * 1000;
  });
};

export default function TrailerStatusApp() {
  const [states, setStates] = useState(() => {
    const initial = {};
    allTors.forEach((id) => {
      initial[id] = {
        status: "Frei",
        timestamp: "",
        comment: "",
        override: "",
        archive: [],
        locked: false,
        showArchive: false,
      };
    });
    return initial;
  });

  const [notification, setNotification] = useState("");
  const [notificationStyle, setNotificationStyle] = useState("bg-green-600 text-white");

  useEffect(() => {
    const interval = setInterval(() => {
      setStates((prev) => {
        const updated = {};
        Object.keys(prev).forEach((id) => {
          updated[id] = {
            ...prev[id],
            archive: removeOldArchiveEntries(prev[id].archive),
          };
        });
        return updated;
      });
    }, 60000);
    return () => clearInterval(interval);
  }, []);

  useEffect(() => {
    if (notification) {
      const timeout = setTimeout(() => setNotification(""), 6000);
      return () => clearTimeout(timeout);
    }
  }, [notification]);

  const showNotification = (message, status = "") => {
    setNotification(message);
    if (status) {
      setNotificationStyle(getNotificationColor(status));
    } else {
      setNotificationStyle("bg-green-600 text-white");
    }
  };

  const handleClick = (id) => {
    if (states[id].locked) return;
    const newStatus = getNextState(states[id].status);
    setStates((prev) => {
      const shouldUpdateTimestamp = [
        "Frei",
        "Leerer Trailer benötigt",
        "Beladung",
        "Fertig",
      ].includes(newStatus);
      const newArchive = shouldUpdateTimestamp
        ? [...prev[id].archive, {
            status: newStatus,
            timestamp: new Date().toLocaleString(),
            comment: prev[id].comment,
          }]
        : prev[id].archive;
      return {
        ...prev,
        [id]: {
          ...prev[id],
          status: newStatus,
          timestamp: shouldUpdateTimestamp ? new Date().toLocaleString() : prev[id].timestamp,
          override: "",
          archive: newArchive,
        },
      };
    });
    showNotification(`Tor ${id}: Status geändert zu ${newStatus}`, newStatus);
  };

  const toggleLock = (id) => {
    if (states[id].status !== "Frei") return;
    setStates((prev) => {
      const locked = !prev[id].locked;
      return {
        ...prev,
        [id]: {
          ...prev[id],
          locked,
          status: locked ? "Anders belegt" : "Frei",
        },
      };
    });
  };

  const handleErledigt = async (id) => {
    const currentState = states[id].status;
    let question = "";

    if (currentState === "Leerer Trailer benötigt") {
      question = "Ist der Trailer am Tor angekommen?";
    } else if (currentState === "Fertig") {
      question = "Ist der Trailer abgeholt?";
    }

    if (question) {
      const confirmed = window.confirm(question);
      if (!confirmed) return;
    }

    setStates((prev) => {
      const s = prev[id];
      let newStatus = s.status;
      let newOverride = s.override;
      let newComment = s.comment;

      if (s.status === "Leerer Trailer benötigt") {
        newStatus = "Beladung";
      } else if (s.status === "Fertig") {
        newStatus = "Frei";
        newOverride = "";
        newComment = "";
      }

      return {
        ...prev,
        [id]: {
          ...s,
          status: newStatus,
          override: newOverride,
          comment: newComment,
          timestamp: new Date().toLocaleString(),
        },
      };
    });
  };

  const renderTorCards = (ids) => (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
      {ids.map((id) => (
        <Card key={id} className={`p-4 shadow-xl ${getBackgroundColor(states[id].status)}`}>
          <CardContent>
            <div className="flex justify-between items-center mb-2">
              <div className="text-lg font-bold">Tor {id}</div>
              <Button className="text-xs px-2 py-1" onClick={() => toggleLock(id)} disabled={states[id].status !== "Frei"}>
                Anders belegt
              </Button>
            </div>
            <div className={`p-4 mb-2 rounded text-center font-semibold ${getStyle(states[id].status, states[id].locked)}`}
                 onClick={() => handleClick(id)}>
              {states[id].status}
            </div>
            <div className="text-sm text-gray-600 mb-2">{states[id].timestamp}</div>
            <textarea
              value={states[id].comment}
              onChange={(e) => setStates((prev) => ({
                ...prev,
                [id]: {
                  ...prev[id],
                  comment: e.target.value,
                },
              }))}
              placeholder="Kommentar..."
              className="w-full p-1 border rounded mb-2"
            />
            {["Leerer Trailer benötigt", "Fertig"].includes(states[id].status) && (
              <>
                <Button
                  className={`w-full mb-1 font-semibold ${states[id].override === "NV" ? "border-2 border-black" : ""}`}
                  variant="outline"
                  onClick={() =>
                    setStates((prev) => ({
                      ...prev,
                      [id]: {
                        ...prev[id],
                        override: prev[id].override === "NV" ? "" : "NV",
                      },
                    }))
                  }
                >
                  NV übernimmt
                </Button>
                <Button
                  className={`w-full mb-1 font-semibold ${states[id].override === "Schlv" ? "border-2 border-black" : ""}`}
                  variant="outline"
                  onClick={() =>
                    setStates((prev) => ({
                      ...prev,
                      [id]: {
                        ...prev[id],
                        override: prev[id].override === "Schlv" ? "" : "Schlv",
                      },
                    }))
                  }
                >
                  Schlv. übernimmt
                </Button>
              </>
            )}
            <Button
              className="w-full mb-1"
              onClick={() => handleErledigt(id)}
            >
              Erledigt
            </Button>
            <Button
              className="w-full text-xs mt-2"
              variant="outline"
              onClick={() =>
                setStates((prev) => ({
                  ...prev,
                  [id]: {
                    ...prev[id],
                    showArchive: !prev[id].showArchive,
                  },
                }))
              }
            >
              Archiv {states[id].showArchive ? "ausblenden" : "anzeigen"}
            </Button>
            {states[id].showArchive && (
              <div className="mt-2 text-sm border-t pt-2">
                {states[id].archive.map((entry, idx) => (
                  <div key={idx} className="mb-1">
                    <strong>{entry.timestamp}</strong>: {entry.status} {entry.comment && `- ${entry.comment}`}
                  </div>
                ))}
              </div>
            )}
          </CardContent>
        </Card>
      ))}
    </div>
  );

  return (
    <div className="p-6">
      {notification && (
        <div className={`fixed bottom-4 left-1/2 transform -translate-x-1/2 px-4 py-2 rounded shadow-lg z-50 ${notificationStyle}`}>
          {notification}
        </div>
      )}
      {renderTorCards(allTors)}
    </div>
  );
}
